## 准备
vagrant + virtualBox 搭建 3 台 centos 虚拟机，安装 docker，并且 3 台虚拟机能够互相访问。

关闭防火墙
```sh
systemctl stop firewalld
systemctl disable firewalld
```

关闭 selinux
```sh
sed -i 's/enforcing/disabled/' /etc/selinux/config
setenforce 0
```

关闭 swap
```sh
# 暂时关闭
swapoff -a

# 永久关闭
# 注释掉 swap 分区项
vim /etc/fstab

# 检查
free -m
```

同步系统时间
```sh
ntpdate time.windows.com
```

添加域名
```sh
# /etc/hosts
192.168.20.11 kube01
192.168.20.12 kube02
192.168.20.13 kube03
```

设置主机名
```sh
hostnamectl set-hostname kube01
```

将桥接的 IPv4 流量传递到 iptables 的链
```sh
# /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1

sysctl --system
```

安装 docker

设置 cgroup 驱动
```sh
# /etc/docker/daemon.json

{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ]
}

mkdir -p /etc/systemd/system/docker.service.d

# Restart Docker
systemctl daemon-reload
systemctl restart docker
```


安装 kubelet, kubeadm, kubectl
```sh
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```

```sh
yum install -y kubelet-1.16.0 kubeadm-1.16.0 kubectl-1.16.0
systemctl enable kubelet
systemctl start kubelet
```

kubelet 问题排查
```sh
# 查看日志
journalctl -fu kubelet.service
journalctl -fu docker.service

# 重新启动
systemctl daemon-reload
systemctl restart kubelet
```


## 部署 master 节点
```sh
# 指定阿里云镜像仓库地址

sudo kubeadm init \
  --apiserver-advertise-address=192.168.20.11 \
  --image-repository registry.aliyuncs.com/google_containers \
  --kubernetes-version v1.16.0 \
  --service-cidr=10.96.0.0/16 \
  --pod-network-cidr=10.244.0.0/16
```

kubectl 工具
```sh
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
```sh
kubectl get nodes
kubectl get cs
```


## 网络插件
```sh
kubectl apply -f kube-flannel.yml
kubectl get pods -n kube-system
```
```sh
docker pull quay.io/coreos/flannel:v0.11.0-amd64
docker pull quay.azk8s.cn/coreos/flannel:v0.11.0-amd64

docker tag quay.azk8s.cn/coreos/flannel:v0.11.0-amd64 quay.io/coreos/flannel:v0.11.0-amd64
```


## 部署 worker 节点
```sh
sudo kubeadm join 192.168.20.11:6443 --token intm8b.2uh8ogkr2p72qsuo \
    --discovery-token-ca-cert-hash sha256:9598f2debea4b62bc0d398a745d512b8047ae86b4515ecf7aaf4af24958efc01
```

默认 token 有效期为 24 小时，当过期之后需要重新创建 token
```sh
kubeadm token create
kubeadm token list

openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'
63bca849e0e01691ae14eab449570284f0c3ddeea590f8da988c07fe2729e924


```


## 测试集群
```sh
kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --port=80 --type=NodePort
```


## 部署 Dashboard
```sh
# 修改为 NodePort 类型
kubectl apply -f dashboard.yaml
kubectl get pods -n kubernetes-dashboard
```

将 kubernetes-dashboard 这个 service account 绑定到 cluster-admin 角色
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kubernetes-dashboard
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: kubernetes-dashboard
    namespace: kubernetes-dashboard
```

根据自签证书生成 secret
```sh
kubectl create secret generic kubernetes-dashboard-certs --from-file=$HOME/certs -n kubernetes-dashboard
```
```yaml
containers:
  - args:
    - --tls-cert-file=/tls.crt
    - --tls-key-file=/tls.key
```

登录 token
```sh
kubectl -n kubernetes-dashboard describe secret <secret>
```


## kubeadm 初始化工作
1. 生成证书：kubernetes 对外提供服务时，除非专门开启不安全模式，否则都要通过 HTTPS 才能访问 kube-apiserver，这就需要为 Kubernetes 集群配置好证书文件。kubeadm 生成的证书文件都放在 Master 节点的 `/etc/kubernetes/pki` 目录下。在这个目录下，最主要的证书文件是 ca.crt 和对应的私钥 ca.key

2. 生成配置文件：为其他组件生成访问 kube-apiserver 所需的配置文件，如 admin.conf，controller-manager.conf，kubelet.conf，scheduler.conf 等

3. 生成 Pod 配置文件：Master 组件的 YAML 文件会被生成在 `/etc/kubernetes/manifests` 下，包括 etcd.yaml，kube-apiserver.yaml，kube-controller-manager.yaml，kube-scheduler.yaml 等

4. 创建组件容器：kubelet 会根据这些 YAML 文件创建 master 组件容器并启动组件容器

5. token：等到组件全部运行起来之后，kubeadm 就会为集群生成一个 bootstrap token。只要持有这个 token，任何一个安装了 kubelet 和 kubadm 的节点，都可以通过 kubeadm join 加入到这个集群当中。kubeadm 会将 ca.crt 等 Master 节点的重要信息，通过 `ConfigMap` 的方式保存在 Etcd 当中，供后续部署 `Node` 节点使用。这个 `ConfigMap` 的名字是 cluster-info

6. 安装默认插件：默认安装 kube-proxy 和 DNS 这两个插件，分别用来提供整个集群的服务发现和 DNS 功能


