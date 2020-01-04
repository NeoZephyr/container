https://github.com/lizhenliang/ansible-install-k8s
https://github.com/lizhenliang/k8s-statefulset/tree/master/etcd


export http_proxy=http://127.0.0.1:50236;export https_proxy=http://127.0.0.1:50236;

ADD index.html ${DOC_ROOT}
ADD entrypoint.sh /bin/
EXPOSE 80/tcp
CMD ["/usr/sbin/nginx", "-g", "daemon off;"]
ENTRYPOINT ["/bin/entrypoint.sh"]

# entrypoint.sh
cat > /etc/nginx/conf.d/www.conf << EOF
server {
    server_name $HOSTNAME;
    listen ${LISTEN_IP:-0.0.0.0}:${LISTEN_PORT:-80};
    root ${DOC_ROOT:-/usr/share/nginx/html};
}
EOF

exec "$@"


所有节点更改
docekr daemon.json 添加可信任地址
insecure-registries

FROM ip/library/tomcat:v1

docker login ip
docker tag java-demo:v1 ip/demo/java-demo:v1

kubectl create deployment java-demo --image=ip/demo/java-demo:v1 -o yaml --dray-run > java-demo.yaml

docker secret 用于拉取镜像
imagePullSecrets
kubectl create secret docker-registry docker-registry-auth --docker-username=admin --docker-password=Harbor12345 --docker-server=ip

kubectl expose deployment java-demo --port=80 --target-port=8080 --type=NodePort -o yaml --dry-run > service.yaml

ingress 对外暴露服务





代理
kubectl proxy --port=8080

Operator
```sh
git clone https://github.com/coreos/etcd-operator
```
为 Etcd Operator 创建 RBAC 规则，因为 Etcd Operator 需要访问 Kubernetes 的 APIServer 来创建对象
```sh
./etcd-operator/example/rbac/create_role.sh
```
```sh
kubectl create -f ./etcd-operator/example/deployment.yaml
```
一旦 Etcd Operator 的 Pod 进入了 Running 状态，就有一个 CRD 被自动创建出来
```sh
kubectl get pods
kubectl get crd
```
创建 etcd 集群
```sh
kubectl apply -f ./etcd-operator/example/example-etcd-cluster.yaml
```

Operator 的工作原理，利用了 Kubernetes 的自定义 API 资源（CRD），来描述我们想要部署的“有状态应用”；然后在自定义控制器里，根据自定义 API 对象的变化，来完成具体的部署和运维工作。

Flannel 支持三种后端实现
1. VXLAN
2. host-gw
3. UDP

宿主机 Node1，有容器 container1，ip 地址为 100.96.1.2，对应 docker0 网桥地址为 100.96.1.1/24
宿主机 Node2，有容器 container2，ip 地址为 100.96.2.3，对应 docker0 网桥地址为 100.96.2.1/24

UDP 模式
container1 里面的进程发送 ip 数据包，源地址为 100.96.1.2，目的地址为 100.96.2.3。由于目的地址不在 Node1 的 docker0 网桥的网段里，因此这个 ip 数据包交给默认路由规则，通过容器的网关进入 docker0 网桥，到达宿主机。此时，ip 数据包的下一个目的地就取决于宿主机上面的路由规则。

查看宿主机路由规则
```sh
ip route
```

由于 ip 数据包的目的地址匹配不到宿主机 docker0 网桥对应的 100.96.1.0/24 网段，只能匹配到 100.96.0.0/16 这条路由规则，从而进入到 flannel0 设备。

flannel0 设备会包 ip 数据包交给这个设备的应用程序，即 Flannel 进程，这是一个从内核态（Linux 操作系统）向用户态（Flannel 进程）的流动方向。

由 Flannel 管理的容器网络里，一台宿主机上的所有容器，都属于该宿主机被分配的一个“子网”。其中，Node 1 的子网是 100.96.1.0/24，Node 2 的子网是 100.96.2.0/24。而这些子网与宿主机的对应关系，保存在 Etcd 中。flanneld 进程根据 ip 数据包的目的 ip 地址匹配到对应的子网，并找到该子网对应的宿主机 ip 地址。flanneld 进程会将该 ip 数据包封装到一个 UDP 包中，发给 Node2。这个 UDP 包的源地址，就是 flanneld 所在的 Node1 的地址，而目的地址，则是 container2 所在的宿主机 Node2 的地址。由于每台宿主机上的 flanneld，都监听着一个 8285 端口，所以 flanneld 只要把 UDP 包发往 Node2 的 8285 端口即可。


Node2 上监听 8285 端口的 flanneld 进程，就可以从这个 UDP 包里解析出封装在里面的原 ip 数据包，然后直接把这个 ip 数据包发送给它所管理的 flannel0 设备，即数据从用户态流向了内核态。根据路由规则，将 ip 数据包转发到 docker0 网桥，最终到达 container2

上述流程有一个重要前提，即 docker0 网桥的地址范围必须是 Flannel 为宿主机分配的子网。
```sh
FLANNEL_SUBNET=100.96.1.1/24

# docker daemon 启动时配置 bip 参数
dockerd --bip=$FLANNEL_SUBNET
```

以上即为 UDP 模式，由于多了一个额外的步骤，即 flanneld 的处理过程，需要多次用户态与内核态之间的数据拷贝，有严重性能问题，已经被废弃了。

VXLAN 模式
VXLAN 即虚拟可扩展局域网。VXLAN 的设计思想是在现有的三层网络之上，“覆盖”一层虚拟的、由内核 VXLAN 模块负责维护的二层网络。VXLAN 会在宿主机上设置一个特殊的网络设备作为“隧道”的两端。这个设备就叫作 VTEP，即虚拟隧道端点。VTEP 的作用与 UDP 中的 flanneld 进程非常相似。每台宿主机上名叫 flannel.1 的设备，就是 VXLAN 所需的 VTEP 设备，它既有 ip 地址，也有 MAC 地址。container1 的 ip 地址为 10.1.15.2，container2 的 ip 地址为 10.1.16.3。container1 发出的 ip 包目的地址是 10.1.16.3，会先到达 docker0 网桥，然后路由到本机 flannel.1 设备进行处理。根据 Flannel 网络路由规则，凡是发往 10.1.16.0/24 网段的 ip 包，都需要经过 flannel.1 设备发出，网关地址是：10.1.16.0。Node1 上面的 flannel.1 设备，即源 VTEP 设备，在收到原始 ip 数据包之后，会在上面加上一个目的 MAC 地址，封装成一个二层数据帧，称为内部数据帧，然后发送给目的 VTEP 设备。接下来，linux 内核将数据帧进一步封装成外部数据帧：在内部数据帧前面加上一个特殊的 VXLAN 头，这个 VXLAN 头里面有一个重要的标志叫做 VNI，它是 VTEP 设备识别某个数据帧是不是应该归自己处理的重要标识，在 Flannel 中，默认值为 1。之后，Linux 内核会把这个数据帧封装进一个 UDP 包里发出去。

Node 1 上的 flannel.1 设备将数据帧从 eth0 网卡发出去之后，到达 Node2 节点的 eth0 网卡。Node 2 的内核网络栈会发现这个数据帧里有 VXLAN Header，且 VNI 的值为 1。于是 Linux 内核会对它进行拆包，获取内部数据帧之后交给 flannel.1 设备。而 flannel.1 设备进一步拆包，取出原始 ip 包，最终到达 container2


以上两种实现容器跨主机网络的方法，用户的容器都是连接在 docker0 网桥上。而网络插件则在宿主机上面创建了一个特殊设备，UDP 创建的是 TUN 设备，VXLAN 创建的是 VTEP 设备。docker0 网桥于这些设备之间，通过 ip 路由表转发协作。因此，网络插件要做的事情，就是把不同宿主机上的特殊设备连通，从而达到容器跨主机通信的目的。

Kubernetes 是通过一个叫作 cni 的接口，维护了一个单独的网桥来代替 docker0，在宿主机上的设备名称默认是 cni0。Kubernetes 之所以要设置这样一个与 docker0 网桥功能几乎一样的 CNI 网桥，主要原因包括两个方面：
1. Kubernetes 项目并没有使用 Docker 的网络模型，所以它并不希望、也不具备配置 docker0 网桥的能力
2. 这与 Kubernetes 如何配置 Pod，也就是 Infra 容器的 Network Namespace 密切相关。Kubernetes 在启动 Infra 容器之后，就可以直接调用 CNI 网络插件，为这个 Infra 容器的 Network Namespace，配置符合预期的网络栈

一个 Network Namespace 的网络栈包括：网卡（Network Interface）、回环设备（Loopback Device）、路由表（Routing Table）和 iptables 规则

查看 CNI 插件基础可执行文件
```sh
ls -al /opt/cni/bin
```
这些可执行文件，按照功能可分为以下三类：
第一类，叫作 Main 插件，它是用来创建具体网络设备的二进制文件
第二类，叫作 IPAM（IP Address Management）插件，它是负责分配 IP 地址的二进制文件
第三类，是由 CNI 社区维护的内置 CNI 插件

以 Flannel 项目为例，要实现一个给 Kubernetes 用的容器网络方案，需要做如下工作：
1. 首先，实现这个网络方案本身，其实就是 flanneld 进程里的主要逻辑
2. 然后，实现该网络方案对应的 CNI 插件，主要做的就是配置 Infra 容器里面的网络栈，并把它连接在 CNI 网桥上

flanneld 启动后会在每台宿主机上生成它对应的 cni 配置文件，也就是一个 ConfigMap，从而告诉 Kubernetes，这个集群要使用 Flannel 作为容器网络方案。cni 配置文件内容如下：
```
cat /etc/cni/net.d/10-flannel.conflist 

{
  "name": "cbr0",
  "plugins": [
    {
      "type": "flannel",
      "delegate": {
        "hairpinMode": true,
        "isDefaultGateway": true
      }
    },
    {
      "type": "portmap",
      "capabilities": {
        "portMappings": true
      }
    }
  ]
}
```
在 Kubernetes 中，处理容器网络相关的逻辑并不会在 kubelet 主干代码里执行，而是会在具体的 CRI（Container Runtime Interface，容器运行时接口）实现里完成。对于 Docker 项目来说，它的 CRI 实现叫作 dockershim。dockershim 会加载上述的 cni 配置文件，并且把列表里的第一个插件、也就是 flannel 插件，设置为默认插件。在后面的执行过程中，flannel 和 portmap 插件会按照定义顺序被调用，从而依次完成“配置容器网络”和“配置端口映射”这两步操作。

cni 插件的工作原理
1. 创建 Pod 时，首先创建的 Infra 容器，由 dockershim 通过调用 Docker API 调用并启动
2. 接着执行一个叫作 SetUpPod 的方法，为 cni 插件准备参数，然后调用 cni 插件为 Infra 容器配置网络

这里要调用的 cni 插件，就是 /opt/cni/bin/flannel，它所需要的参数，分为两部分
第一部分，是由 dockershim 设置的一组 cni 环境变量，最重要的环境变量参数叫作：CNI_COMMAND。它的取值只有两种：ADD 和 DEL。ADD 和 DEL 操作，就是 cni 插件唯一需要实现的两个方法。ADD 的含义是：把容器添加到 cni 网络里；DEL 操作的含义则是：把容器从 cni 网络里移除掉。以 ADD 操作为例，需要的参数包括容器里网卡的名字 eth0（CNI_IFNAME）、Pod 的 Network Namespace 文件的路径（CNI_NETNS）、容器的 ID（CNI_CONTAINERID）等。

第二部分，则是 dockershim 从 CNI 配置文件里加载到的、默认插件的配置信息


host-gw 模式
假设 Node1 上有 container1，Node2 上有 container2。Node1 ip 地址为 10.168.0.2，container1 的 ip 地址为 10.244.0.2；Node2 ip 地址为 10.168.0.3，container2 的 ip 地址为 10.244.1.3。
当设置 Flannel 使用 host-gw 模式之后，flanneld 会在宿主机上创建对应规则。如在 Node1 上面创建 `10.244.1.0/24 via 10.168.0.3 dev eth0`，即表示目的 ip 地址属于 10.244.1.0/24 网段的 ip 包，应该经过本机的 eth0 设备发出去，并且下一跳地址是 10.168.0.3。可以看到，对应的下一跳地址对应的正是我们的目的宿主机 Node2。当 Node2 的内核网络栈从二层数据帧里拿到 ip 包后，查看 ip 包中的目的地址为 10.244.1.3，即 container2 的地址，于是根据路由表中 10.244.1.0/24 对应的路由规则，从而进入 cni0 网桥，从而进入 container2。

通过以上可见，host-gw 模式的工作原理，其实就是将每个 Flannel 子网的“下一跳”，设置成了该子网对应的宿主机的 ip 地址，即这台“主机”（Host）会充当这条容器通信路径里的“网关”（Gateway）。在这种模式下，容器通信的过程就免除了额外的封包和解包带来的性能损耗，相比其他网络方案所带来的性能损失百分比要少。

host-gw 模式能够正常工作的核心，就在于 ip 包在封装成帧发送出去的时候，会使用路由表里的“下一跳”来设置目的 MAC 地址。这样，它就会经过二层网络到达目的宿主机。因此，host-gw 模式必须要求集群宿主机之间是二层连通的。

Calico 项目提供的网络解决方案，与 Flannel 的 host-gw 模式，几乎是完全一样。也就是说，Calico 也会在每台宿主机上，添加一个格式如下所示的路由规则：
```
< 目的容器 ip 地址段 > via < 网关的 ip 地址 > dev eth0
```
其中，网关的 ip 地址，正是目的容器所在宿主机的 ip 地址

不同于 Flannel 通过 Etcd 和宿主机上的 flanneld 来维护路由信息的做法，Calico 项目使用 BGP 来自动地在整个集群中分发路由信息。BGP 全称为 Border Gateway Protocol，即边界网关协议。边界网关协议中，负责把自治系统连接在一起的路由器，称为边界网关。它跟普通路由器的不同之处在于，它的路由表里拥有其他自治系统里的主机路由信息。使用 BGP 之后，在每个边界网关上都会运行着一个小程序，它们会将各自的路由表信息，通过 TCP 传输给其他的边界网关。而其他边界网关上的这个小程序，则会对收到的这些数据进行分析，然后将需要的信息添加到自己的路由表里。所以说，BGP 就是在大规模网络中实现节点路由信息共享的一种协议。而 BGP 的这个能力，正好可以取代 Flannel 维护主机上路由表的功能。

Calico 项目的架构由三个部分组成：
1. Calico 的 CNI 插件，与 Kubernetes 对接
2. Felix，是一个 DaemonSet，负责在宿主机上插入路由规则，以及维护 Calico 所需的网络设备等工作
3. BIRD，即 BGP 的客户端，专门负责在集群里分发路由规则信息

除了对路由信息的维护方式之外，Calico 项目与 Flannel 的 host-gw 模式的另一个不同之处，就是它不会在宿主机上创建任何网桥设备。

Calico 的 CNI 插件会为每个容器设置一个 Veth Pair 设备，然后把其中的一端放置在宿主机上。由于 Calico 没有使用 CNI 的网桥模式，Calico 的 CNI 插件还需要在宿主机上为每个容器的 Veth Pair 设备配置一条路由规则，用于接收传入的 IP 包。比如，宿主机 Node 2 上的 Container 4 对应的路由规则，如下所示：
```
10.233.2.3 dev cali5863f3 scope link
```
表示发往 10.233.2.3 的 IP 包，应该进入 cali5863f3 设备

容器发出的 IP 包就会经过 Veth Pair 设备出现在宿主机上。然后，宿主机网络栈就会根据路由规则的下一跳 IP 地址，把它们转发给正确的网关。

Calico 维护的网络在默认配置下，使用 Node-to-Node Mesh 的模式。每台宿主机上的 BGP Client 都需要跟其他所有节点的 BGP Client 进行通信以便交换路由信息。随着节点数量 N 的增加，这些连接的数量就会以 N²的规模快速增长，从而给集群本身的网络带来巨大的压力。因此，Node-to-Node Mesh 模式一般推荐用在少于 100 个节点的集群里。而在更大规模的集群中，需要使用 Route Reflector 模式。

Route Reflector 模式下，Calico 会指定一个或者几个专门的节点，也就是所谓的 Route Reflector 节点，来负责跟所有节点建立 BGP 连接从而学习到全局的路由规则。而其他节点，只需跟这几个专门的节点交换路由信息，就可以获得整个集群的路由规则信息了。这样 BGP 连接的规模控制在 N 数量级上。

Flannel host-gw 模式最主要的限制，就是要求集群宿主机之间是二层连通的。而这个限制对于 Calico 来说而言也存在。

假如有两台处于不同子网的宿主机 Node1 和 Node2，对应的 ip 地址分别为 192.168.1.2 和 192.168.2.2。由于这两台机器通过路由器实现了三层转发，因此这两个 ip 地址之间是可以相互通信的。现在，位于 Node1 节点的 container1 想要访问 Node2 节点的 container4，Calico 会尝试在 Node1 上添加如下路由规则：
```
10.233.2.0/16 via 192.168.2.2 eth0
```
这里规则里面下一跳的地址为 192.168.2.2，但是该地址与 Node1 不在同一个子网，没有办法通过二层网络将 ip 包转发给下一跳地址。

要解决这个问题，需要为 Calico 打开 IPIP 模式。在该模式下，Felix 进程在 Node 1 上添加的路由规则如下：
```
10.233.2.0/24 via 192.168.2.2 tunl0
```
虽然规则中下一跳地址仍为 192.168.2.2，即 Node2 的 ip 地址，但这次负责将 ip 包发出去的设备由 eth0 变为了 tunl0。tunl0 设备是一个 ip 隧道设备，ip 包进入进入 ip 隧道设备之后，会被 Linux 内核的 IPIP 驱动处理。IPIP 驱动会将这个 ip 包直接封装在一个宿主机网络的 ip 包中，即原 ip 包成为新 ip 包的 payload，并加上新的 ip header，新 ip 包的目的地址正是原 ip 包的下一跳地址。这样，原先从容器到 Node 2 的 ip 包，就被伪装成了一个从 Node 1 到 Node 2 的 ip 包。由于宿主机之间已经使用路由器配置了三层转发，这个 ip 包在离开 Node1 之后可以经过路由器，最终到达 Node2。

从以上分析可以看到，使用 IPIP 模式之后，集群的网络性能会因为额外的封包和解包工作而下降。所以，实际使用中，尽量将所有宿主机放到一个子网，从而避免使用 IPIP 模式。

如果 Calico 项目能够让宿主机之间的路由设备，即网关，也通过 BGP 协议学习到 Calico 网络里的路由规则，那么从容器发出的 ip 包，就可以通过这些设备路由到目的宿主机。
比如在 Node1 节点上添加路由规则：
```
10.233.2.0/24 via 192.168.1.1 eth0
```
在 Node1 节点对应网关 Router1（192.168.1.1）上添加路由规则：
```
10.233.2.0/24 via 192.168.2.1 eth0
```
那么 Container1 发出的 ip 包，就可以通过两次下一跳，到达 Node2 的网关 Router2（192.168.2.1）上面。依次类推，继续在 Router2 上面添加一条路由规则，最终将 ip 包转发到 Node2 上面。以上流程虽然简单明了，但是在公有云场景不可行，这是由于公有云环境下，宿主机之间的网关是不会允许用户进行干预和设置的。不过，在私有部署环境下，宿主机属于不同子网（VLAN）十分常见，我们需要将宿主机网关也加入到 BGP Mesh 里从而避免使用 IPIP。

在 Calico 项目中，提供了两种将宿主机网关设置成 BGP Peer 的解决方案
第一种方案，所有宿主机都跟宿主机网关建立 BGP Peer 关系。这种方案下，Node 1 和 Node 2 就需要主动跟宿主机网关 Router 1 和 Router 2 建立 BGP 连接。从而将类似于 10.233.2.0/24 这样的路由信息同步到网关上去。

这种方式下，Calico 要求宿主机网关必须支持一种叫作 Dynamic Neighbors 的 BGP 配置方式。这是因为，在常规的路由器 BGP 配置里，运维人员必须明确给出所有 BGP Peer 的 IP 地址。考虑到 Kubernetes 集群可能会有成百上千个宿主机，而且还会动态地添加和删除节点，这时候再手动管理路由器的 BGP 配置就非常麻烦了。而 Dynamic Neighbors 则允许你给路由器配置一个网段，然后路由器就会自动跟该网段里的主机建立起 BGP Peer 关系。

第二种方案，使用一个或多个独立组件负责搜集整个集群里的所有路由信息，然后通过 BGP 协议同步给网关。而在大规模集群中，Calico 本身就推荐使用 Route Reflector 节点的方式进行组网。所以，这里负责跟宿主机网关进行沟通的独立组件，直接由 Route Reflector 兼任即可。

更重要的是，这种情况下网关的 BGP Peer 个数是有限并且固定的。所以我们就可以直接把这些独立组件配置成路由器的 BGP Peer，而无需 Dynamic Neighbors 的支持。这些独立组件的工作原理也很简单：它们只需要 WATCH Etcd 里的宿主机和对应网段的变化信息，然后把这些信息通过 BGP 协议分发给网关即可。



Kubernetes 里的 Pod 默认可以接收来自任何发送方的请求，或者向任何接收方发送请求。如果需要作出限制，就必须通过 NetworkPolicy 对象来指定。

podSelector 字段定义 NetworkPolicy 的限制范围，如果该字段为空，那么这个 NetworkPolicy 就会作用于当前 Namespace 下的所有 Pod。一旦 pod 被 NetworkPolicy 选中，那么这个 Pod 就会进入拒绝所有（Deny All）的状态，也就是说这个 Pod 既不允许被外界访问，也不允许对外界发起访问。可以在 NetworkPolicy 中定义白名单规则。在 policyTypes 字段，定义这个 NetworkPolicy 的类型是 ingress 和 egress，也就是说会影响流入（ingress）请求，也会影响流出（egress）请求。ingress 字段中定义了 from 和 ports，表示允许流入的白名单和端口，允许流入的白名单可以使用 ipBlock，namespaceSelector 和 podSelector。egress 中的字段与 ingress 类似。
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: test-network-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - ipBlock:
        cidr: 172.17.0.0/16
        except:
        - 172.17.1.0/24
    - namespaceSelector:
        matchLabels:
          project: myproject
    - podSelector:
        matchLabels:
          role: frontend
    ports:
    - protocol: TCP
      port: 6379
  egress:
  - to:
    - ipBlock:
        cidr: 10.0.0.0/24
    ports:
    - protocol: TCP
      port: 5978
```
需要注意的是，在定义白名单部分的时候，namespaceSelector 和 podSelector 是或的关系，也就是说无论是 Namespace 满足条件，还是 Pod 满足条件，这个 NetworkPolicy 都会生效。但如果定义成为如下规则，就会是与的关系，必须同时满足条件
```
...
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          user: alice
      podSelector:
        matchLabels:
          role: client
  ...
```

综上，这个 NetworkPolicy 对象，指定的隔离规则如下所示：
1. 该隔离规则只对 default Namespace 下的，携带了 role=db 标签的 Pod 有效。限制的请求类型包括 ingress（流入）和 egress（流出）。
2. Kubernetes 会拒绝任何访问被隔离 Pod 的请求，除非这个请求来自于以下“白名单”里的对象，并且访问的是被隔离 Pod 的 6379 端口。这些“白名单”对象包括：
3. default Namespace 里的，携带了 role=fronted 标签的 Pod；
4. 任何 Namespace 里的、携带了 project=myproject 标签的 Pod；
5. 任何源地址属于 172.17.0.0/16 网段，且不属于 172.17.1.0/24 网段的请求
6. Kubernetes 会拒绝被隔离 Pod 对外发起任何请求，对外发起任何请求，除非请求的目的地址属于 10.0.0.0/24 网段，并且访问的是该网段地址的 5978 端口。


如果要使用 NetworkPolicy，需要 CNI 网络插件的支持。目前已经实现了 NetworkPolicy 的网络插件包括 Calico、Weave 和 kube-router 等多个项目，但是并不包括 Flannel 项目。

```
apiVersion: extensions/v1beta1
kind: NetworkPolicy
metadata:
  name: test-network-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      role: db
  ingress:
   - from:
     - namespaceSelector:
         matchLabels:
           project: myproject
     - podSelector:
         matchLabels:
           role: frontend
     ports:
       - protocol: tcp
         port: 6379
```
Kubernetes 网络插件对 Pod 进行隔离，其实是靠在宿主机上生成 NetworkPolicy 对应的 iptable 规则来实现的。设置好隔离规则之后，网络插件将所有对被隔离 Pod 的访问请求，都转发到规则上去进行匹配。并且，如果匹配不通过，这个请求应该被拒绝。CNI 网络插件通过设置两组 iptables 规则来实现
第一组规则，负责“拦截”对被隔离 Pod 的访问请求

iptables 只是一个操作 Linux 内核 Netfilter 子系统的界面。也就是说，Netfilter 子系统的作用，就是 Linux 内核里挡在网卡和用户态进程之间的一道防火墙。在 ip 包进出的路径上有几个关键的检查点，称之为链。

当一个 IP 包通过网卡进入主机之后，它就进入了 Netfilter 定义的流入路径（Input Path）里。
在这个路径中，IP 包要经过路由表路由来决定下一步的去向。而在路由之前，Netfilter 设置了一个名叫 PREROUTING 的“检查点”。

经过路由之后，IP 包的去向就分为了两种：
第一种，继续在本机处理，这时候，IP 包将继续向上层协议栈流动。在它进入传输层之前，Netfilter 会设置一个名叫 INPUT 的“检查点”。到这里，IP 包流入路径（Input Path）结束。接下来，这个 IP 包通过传输层进入用户空间，交给用户进程处理。而处理完成后，用户进程会通过本机发出返回的 IP 包。这时候，这个 IP 包就进入了流出路径（Output Path）。此时，IP 包首先还是会经过主机的路由表进行路由。路由结束后，Netfilter 就会设置一个名叫 OUTPUT 的“检查点”。然后，在 OUTPUT 之后，再设置一个名叫 POSTROUTING “检查点”。

第二种，被转发到其他目的地，这时候 ip 包不会进入传输层，而是会继续在网络层流动，从而进入到转发路径（Forward Path）。在转发路径中，Netfilter 会设置一个名叫 FORWARD 的“检查点”。而在 FORWARD“检查点”完成后，IP 包就会来到流出路径。而转发的 IP 包由于目的地已经确定，它就不会再经过 OUTPUT，而是会直接来到 POSTROUTING 检查点。因此，POSTROUTING 的作用，其实就是上述两条路径，最终汇聚在一起的“最终检查点”。


Kubernetes 调度器
默认调度器的主要职责，就是为一个新创建出来的 Pod，寻找一个最合适的节点

1. 默认调度器会首先调用一组叫作 Predicate 的调度算法，来检查每个 Node
2. 调用一组叫作 Priority 的调度算法，来给上一步得到的结果里的每个 Node 打分。最终的调度结果，就是得分最高的那个 Node

Kubernetes 的调度器的核心，实际上就是两个相互独立的控制循环
第一个控制循环，我们可以称之为 Informer Path。启动一系列 Informer，用来监听（Watch）Etcd 中 Pod、Node、Service 等与调度相关的 API 对象的变化。当一个待调度 Pod 被创建出来之后，调度器就会通过 Pod Informer 的 Handler，将这个待调度 Pod 添加进调度队列。此外，Kubernetes 的默认调度器还要负责对调度器缓存（即：scheduler cache）进行更新。

第二个控制循环，是调度器负责 Pod 调度的主循环，称之为 Scheduling Path。Scheduling Path 的主要逻辑，就是不断地从调度队列里出队一个 Pod。然后，调用 Predicates 算法进行“过滤”。这一步“过滤”得到的一组 Node，就是所有可以运行这个 Pod 的宿主机列表。Predicates 算法需要的 Node 信息，都是从 Scheduler Cache 里直接拿到的，这样能保证算法执行效率。接下来，调度器就会再调用 Priorities 算法为上述列表里的 Node 打分，得分最高的 Node，就会作为这次调度的结果。

调度算法执行完成后，调度器就需要将 Pod 对象的 nodeName 字段的值，修改为上述 Node 的名字。这个步骤在 Kubernetes 里面被称作 Bind。

为了不在关键调度路径里远程访问 APIServer，Kubernetes 的默认调度器在 Bind 阶段，只会更新 Scheduler Cache 里的 Pod 和 Node 的信息。这种基于“乐观”假设的 API 对象更新方式，在 Kubernetes 里被称作 Assume。Assume 之后，调度器才会异步地向 APIServer 发起更新 Pod 的请求，来真正完成 Bind 操作。当一个新的 Pod 完成调度需要在某个节点上运行起来之前，该节点上的 kubelet 还会通过一个叫作 Admit 的操作来再次验证该 Pod 是否确实能够运行在该节点上。

Predicates
Predicates 在调度过程中的作用，类似为 Filter。它按照调度策略，从当前集群的所有节点中，过滤出一系列符合条件的节点。这些节点，都是可以运行待调度 Pod 的宿主机。

在 Kubernetes 中，默认的调度策略有以下三种：
GeneralPredicates 负责最基础的调度策略，通过一组过滤条件考察一个 Pod 能不能运行在一个 Node 上。PodFitsResources 检查 Pod 的 requests 字段，计算宿主机的 CPU 和内存资源等是否够用；PodFitsHost 检查宿主机的名字是否跟 Pod 的 spec.nodeName 一致；PodFitsHostPorts 检查 Pod 申请的宿主机端口（spec.nodePort）是不是跟已经被使用的端口有冲突；PodMatchNodeSelector 检查 Pod 的 nodeSelector 或者 nodeAffinity 指定的节点，是否与待考察节点匹配。

之前提到，kubelet 在启动 Pod 前，会执行一个 Admit 操作来进行二次确认。这里二次确认的规则，就是执行一遍 GeneralPredicates。

Volume 相关的过滤规则，负责的是跟容器持久化 Volume 相关的调度策略。NoDiskConflict 检查多个 Pod 声明挂载的持久化 Volume 是否有冲突；MaxPDVolumeCountPredicate 检查一个节点上某种类型的持久化 Volume 是不是已经超过了一定数目；VolumeZonePredicate 检查持久化 Volume 的 Zone（高可用域）标签，是否与待考察节点的 Zone 标签相匹配；VolumeBindingPredicate 负责检查 Pod 对应的 PV 的 nodeAffinity 字段，是否跟某个节点的标签相匹配，如果 Pod 的 PVC 还没有跟具体的 PV 绑定，调度器还要负责检查所有待绑定 PV，当有可用的 PV 存在并且该 PV 的 nodeAffinity 与待考察节点一致时，这条规则才会过滤通过。
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: example-local-pv
spec:
  capacity:
    storage: 500Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-storage
  local:
    path: /mnt/disks/vol1
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - my-node
```
这个 PV 对应的持久化目录，只会出现在名叫 my-node 的宿主机上。所以，任何一个通过 PVC 使用这个 PV 的 Pod，都必须被调度到 my-node 上才可以正常工作。

宿主机相关的过滤规则，要考察待调度 Pod 是否满足 Node 本身的某些条件。例如，PodToleratesNodeTaints 负责检查的就是我们前面经常用到的 Node 的污点机制。只有当 Pod 的 Toleration 字段与 Node 的 Taint 字段能够匹配的时候，这个 Pod 才能被调度到该节点上；NodeMemoryPressurePredicate 检查的是当前节点的内存是不是已经不够充足，如果不充足，那么待调度 Pod 就不能被调度到该节点上。

Pod 相关的过滤规则，这组规则跟 GeneralPredicates 大多数是重合的。比较特殊的是 PodAffinityPredicate，负责检查待调度 Pod 与 Node 上的已有 Pod 之间的亲密（affinity）和反亲密（anti-affinity）关系
```
apiVersion: v1
kind: Pod
metadata:
  name: with-pod-antiaffinity
spec:
  affinity:
    podAntiAffinity: 
      requiredDuringSchedulingIgnoredDuringExecution: 
      - weight: 100  
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: security 
              operator: In 
              values:
              - S2
          topologyKey: kubernetes.io/hostname
  containers:
  - name: with-pod-affinity
    image: docker.io/ocpqe/hello-pod
```
其中，podAntiAffinity 规则指定了这个 Pod 不希望跟任何携带 security=S2 标签的 Pod 存在于同一个 Node 上。PodAffinityPredicate 是有作用域的，示例中的仅对携带了 Key 是 kubernetes.io/hostname标签的 Node 有效。podAffinity 与 podAntiAffinity 相反。

上面这四种类型的 Predicates，就构成了调度器确定一个 Node 可以运行待调度 Pod 的基本策略。
在具体执行时，当开始调度一个 Pod 时，Kubernetes 调度器会同时启动 16 个 Goroutine，来并发地为集群里的所有 Node 计算 Predicates，最后返回可以运行这个 Pod 的宿主机列表。

在 Predicates 阶段完成了节点的过滤之后，Priorities 阶段开始为这些节点打分，得分最高的节点就是最后被 Pod 绑定的最佳节点。Priorities 中有以下常见打分规则：
LeastRequestedPriority，其计算方法可以简单的通过如下公式总结。可以看出，这个算法实际上就是在选择空闲资源（CPU 和 Memory）最多的宿主机。
```
score = (cpu((capacity-sum(requested))10/capacity) + memory((capacity-sum(requested))10/capacity))/2
```
与 LeastRequestedPriority 一起发挥作用的还有 BalancedResourceAllocation，其计算公式如下：
```
score = 10 - variance(cpuFraction,memoryFraction,volumeFraction)*10
```
每种资源的 Fraction 的定义是 ：Pod 请求的资源 / 节点上的可用资源。而 variance 算法的作用，则是计算每两种资源 Fraction 之间的“距离”。而最后选择的，则是资源 Fraction 差距最小的节点。因此，BalancedResourceAllocation 选择的是调度完成后，所有节点里各种资源分配最均衡的那个节点，从而避免一个节点上 CPU 被大量分配、而 Memory 大量剩余的情况。

此外，还有 NodeAffinityPriority、TaintTolerationPriority 和 InterPodAffinityPriority 这三种 Priority。

在默认 Priorities 里，还有一个叫作 ImageLocalityPriority 的策略。如果待调度 Pod 需要使用的镜像很大，并且已经存在于某些 Node 的得分就会比较高。当然，如果大镜像分布的节点数目很少，那么这些节点的权重就会被调低，避免引起调度堆叠的风险。


在实际的执行过程中，调度器里关于集群和 Pod 的信息都已经缓存化，所以 Predicates 和 Priorities 算法的执行过程还是比较快的。

除了以上提到的这些规则外，Kubernetes 调度器里其实还有一些默认不会开启的策略，还可以通过为 kube-scheduler 指定一个配置文件或者创建一个 ConfigMap ，来配置哪些规则需要开启、哪些规则需要关闭。
