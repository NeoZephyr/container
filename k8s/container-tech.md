## 容器与虚拟机
1. 容器启动速度秒级，虚拟机启动速度是分钟级别
2. 容器运行性能接近原生，虚拟机有性能损失
3. 容器占用空间小，可运行成百上千个；虚拟机占用空间大，可运行个数少
4. 容器隔离级别是进程级别，虚拟机隔离级别是系统级别
5. 管理虚拟机 = 管理基础设施，管理容器 = 管理应用本身

## 容器技术
一个正在运行的容器，其实就是一个启用了多个 Linux Namespace 的应用进程。这个进程视图被隔离，使用资源受 Cgroups 配置的限制

### Cgroups
主要作用是限制一个进程组能够使用的资源上限，包括 CPU、内存、磁盘、网络带宽等等。在 Linux 操作系统中，Cgroups 给用户暴露出来的操作接口是文件系统，即它以文件和目录的方式组织在操作系统的 `/sys/fs/cgroup` 路径下

```sh
mount -t cgroup
```

在新创建的 container 目录下，会自动生成该子系统对应的资源限制文件
```sh
# 查看 cpu 限制接口对应的文件路径
cd /sys/fs/cgroup/cpu
mkdir container
```
此时，在后台运行如下命令，通过 `top` 命令得知此时 cpu 占有率达到 100%，进程号为 256
```sh
while true ; do : ; done &
```
此时查看 container 目录下的文件，container 控制组里的 cpu quota 没有任何限制，所以之前的程序执行时才使得 cpu 占有率达到了 100%。cpu period 默认为 100ms
```sh
cat /sys/fs/cgroup/cpu/container/cpu.cfs_quota_us
cat /sys/fs/cgroup/cpu/container/cpu.cfs_period_us
```

现在试图更新 cpu quota 限制，并将被限制的进程的 pid 写入 container 目录里面的 tasks 文件，这样之前的配置就会生效
```sh
echo 20000 > /sys/fs/cgroup/cpu/container/cpu.cfs_quota_us
```
```sh
echo 256 > /sys/fs/cgroup/cpu/container/tasks
```
进行以上限制之后，执行 `top` 命令查看 cpu 占用降为 20%

综上实验结果，对于 docker 而言，只需要在每个子系统下面，为每个容器创建一个控制组（即创建一个新目录），然后在启动容器进程之后，把这个进程的 PID 填写到对应控制组的 tasks 文件中就可以了。
```sh
# 启动容器时，指定参数
docker run -it --cpu-period=100000 --cpu-quota=20000 ubuntu /bin/bash
```

### Namespace
包括各种 Namespace，主要作用是隔离。其中 Mount Namespace 跟其他 Namespace 的使用略有不同的地方：它对容器进程视图的改变，一定是伴随着挂载操作（mount）才能生效。

在 Linux 系统中，一个进程的每种 Linux Namespace，都在它对应的 `/proc/[进程号]/ns` 下有一些对应的虚拟文件，这些虚拟文件会链接到对应的真实的 Namespace 文件上。而一个进程，可以选择加入到某个进程已有的 Namespace 当中，从而达到进入这个进程所在容器的目的，这正是 `docker exec` 的实现原理

#### UTS 主机名和域名

#### Mount 文件系统

#### IPC 消息队列、共享内存

#### PID 进程号

#### User 用户和用户组

#### Network 网络
```sh
ip netns list

# 删除
ip netns delete <网络命名空间>

# 添加
ip netns add <网络命名空间>
```
```sh
ip netns exec <网络命名空间> ip a
ip netns exec <网络命名空间> ip link
```
```sh
# lo 端口 up
ip netns exec <网络命名空间> ip link set dev lo up
```

网络连接
```sh
# 增加一对网络端口
ip link add veth-eth1 type veth peer name veth-eth2

# 为端口设置网络命名空间
ip link set veth-eth1 netns test1
ip link set veth-eth2 netns test2

ip netns exec test1 ip link

# 为端口分配 ip 地址，只能在本地访问
ip netns exec test1 ip addr add 192.168.1.1/24 dev veth-eth1
ip netns exec test2 ip addr add 192.168.1.2/24 dev veth-eth2

# 启用端口
ip netns exec test1 ip link set dev veth-eth1 up
ip netns exec test2 ip link set dev veth-eth2 up

# 在 test1 网络命名空间 ping test2
ip netns exec test1 ping 192.168.1.2
```

docker 多机通信
VXLAN
overlay

### rootfs
对 Docker 容器来说，它最核心的原理实际上就是为待创建的用户进程：
1. 启用 Linux Namespace 配置
2. 设置指定的 Cgroups 参数
3. 切换进程的根目录（Change Root）

挂载在容器根目录上、用来为容器进程提供隔离后执行环境的文件系统，就是所谓的“容器镜像”。专业名称叫作：rootfs（根文件系统）。docker 在镜像设计中，引入分层，而分层利用到了联合文件系统（Union File System），能将多个不同位置的目录联合挂载（union mount）到同一个目录下。即镜像的层都放置在 `/var/lib/docker/aufs/diff` 下的分散目录下，然后被联合挂载在 `/var/lib/docker/aufs/mnt` 里面。

rootfs 由如下三部分组成：
1. 只读层
在容器的 rootfs 最下面几层，挂载方式是只读的
2. 可读写层
在容器的 rootfs 最上面一层，挂载方式是可读写的
3. init 层
在只读层与可读写层之间，由 docker 生成专门用来存放 `/etc/hosts`, `/etc/resolv.conf` 等信息。保证用户执行 `docker commit` 操作时，只会提交可读写层，而不会提交这些由 docker 自动生成的内容

而由于使用了联合文件系统，你在容器里对镜像 rootfs 所做的任何修改，都会被操作系统先复制到这个可读写层，然后再修改。这就是所谓的：Copy-on-Write

Volume 机制，允许你将宿主机上指定的目录或者文件，挂载到容器里面进行读取和修改操作。