## 安装
```sh
# 安装必要的一些系统工具
sudo yum install -y yum-utils device-mapper-persistent-data lvm2

# 添加软件源信息
sudo yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

# 更新并安装 Docker-CE
sudo yum makecache fast

sudo yum list docker-ce --showduplicates
sudo yum -y install docker-ce-18.09.9-3.el7
sudo yum -y install docker-ce

sudo systemctl start docker
sudo systemctl enable docker

sudo usermod -aG docker vagrant
```

## 镜像
镜像加速
```sh
# /etc/docker/daemon.json
curl -sSL https://get.daocloud.io/daotools/set_mirror.sh | sh -s http://f1361db2.m.daocloud.io
```
```
https://dockerhub.azk8s.cn
https://rvthv4hk.mirror.aliyuncs.com
```
```
https://www.kancloud.cn/willseecloud/docker-handbook/1324246
```

查看镜像
```sh
docker image ls
docker images
```

删除镜像
```sh
docker image rm <imageId>
docker rmi <imageId>
```

拉取镜像
```sh
docker pull nginx
```

打标签
```sh
docker tag <imageId> pain/nginx:v1.0
```

启动镜像
```sh
docker run nginx

# 通过中间镜像调试
docker run -it nginx bash

# --name 为容器命名
# --rm 容器退出就删除
# --network 指定容器网络驱动
docker run --name nginx -it --network bridge --rm nginx

# -p 端口映射
docker run -p <hostPort>:<containerPort> nginx
docker run -p <hostIp>:<hostPort>:<containerPort> nginx

# 创建守护式容器
docker run -d nginx
```

查看镜像构建历史
```sh
docker history <imageId>
```

## 容器

查看容器
```sh
docker container ls
docker container ls -a

# 列出所有容器 id
docker container ls -aq

# 查看端口映射
docker port <containerName>
```

启动容器
```sh
docker start <containerId>
```

停止容器
```sh
docker stop <containerId>
```

删除容器
```sh
docker rm <containerId>

# 删除所有容器
docker rm $(docker container ls -aq)

# 删除所有退出状态容器
docker rm $(docker container ls -f "status=exited" -q)
```

容器内运行
```sh
docker exec -it <containerId> bash
```

查看容器运行状态
```sh
# 获取容器日志
docker logs <containerId>
docker logs -f <containerId>
docker logs -ft <containerId>

docker ps -a

# 查看容器内进程
docker top <containerId>
```

查看容器详情
```sh
# 获取配置信息
docker container inspect <containerId>
```

## 持久化
### volume
```sh
docker volume ls
```
```sh
docker volume rm <volumeId>
```
```sh
docker volume inspect <volumeId>
```
```sh
# /var/lib/docker/volumes
# 产生 html volume
docker run -d -v html:/var/web/html --name nginx nginx
```

### 目录映射
```sh
# 目录映射
# bind mounting
docker run -d -v $(pwd):/var/web/html --name nginx nginx
```

### tmpfs
挂载存储在主机系统的内存中，而不会写入主机的文件系统。如果不希望将数据持久存储在任何位置，可以使用 tmpfs，同时避免写入容器可写层提高性能

## 网络
创建容器会同时创建一个独立的网络命名空间，容器之间通过 `docker0` 进行通信

### `bridge`
```sh
# 查看 docker 网络
docker network ls

# 用户自定义 bridge
# 创建 network
docker network create -d bridge my-bridge

docker network inspect <network id>
```

```sh
yum install bridge-utils
brctl show
```

```sh
# 容器之间 link
docker run -d --name test2 --link test1 hello-world

# 容器之间通过 network link
docker run -d --name test3 --network my-bridge hello-world

# 将容器 test2 连接到新网络
# 此时 test2 与 test3 连接到用户创建的 bridge 上面，可以通过 ping <containerName> 验证
docker network connect my-bridge test2
```

### `none`
```sh
# 没有 ip 跟 mac 地址，只能在本地访问
docker run -d --name test1 --network none hello-world
```
```sh
docker network inspect none
```

### `host`
```sh
# 没有独立网络命名空间，跟主机共享网络命名空间
docker run -d --name test1 --network host hello-world
```
```sh
docker network inspect host
```