## service 功能
1. pod 的 ip 不固定
2. 通过 label selector 关联一组 pod，实现负载均衡（4 层）
3. 被 selector 选中的 pod，称为 service 的 Endpoints。只有处于 Running 状态，且 readinessProbe 检查通过的 pod，才会出现在 Endpoints 列表里。可以通过如下命令查看
```sh
kubectl get endpoints <service name>
```


## 暴露服务
### ClusterIp
提供一个内部 ip，只能集群内部访问
```sh
kubectl expose pod app-pod
kubectl expose deployment app-deployment
```

访问方式：
1. 通过访问该虚拟 ip，把请求转发到该 service 所代理的某一个 pod 上
2. 访问 `<service name>.<namespace>.svc.cluster.local` 就可以访问到对应 service 所代理的一个 pod

DNS 服务监视 Kubernetes API，为每一个 service 创建 DNS A 记录用于域名解析。正常情况下，访问 `<service name>.<namespace>.svc.cluster.local` 所解析到的是对应 service 的虚拟 ip。但如果使用的是 Headless Service（clusterIP 指定为 None），访问 `<service name>.<namespace>.svc.cluster.local` 所解析到的是对应 service 代理的某一个 pod 的 ip 地址，即可以直接以 DNS 记录的方式解析出被代理 pod 的 ip 地址

### NodePort
提供一个内部 ip，并在每个节点上启用一个端口来暴露服务，在集群外部通过节点访问服务
```sh
kubectl expose deployment app-deployment --port=80 --type=NodePort --target-port=80 --name=app-service
```

```sh
kubectl get node -o wide
kubectl get node -o yaml

kubectl get node --show-labels
kubectl label node <node> <key>=<value>
kubectl label node <node> <key>-
```

需要注意的是，在 NodePort 方式下，Kubernetes 会在 ip 包离开宿主机发往目的 pod 时，对这个 ip 包做一次 SNAT 操作，将这个 ip 包的源地址替换成了这台宿主机上的 CNI 网桥地址，或者宿主机本身的 ip 地址（如果 CNI 网桥不存在的话）。这么做的原因如下：

当外部的客户端通过 node2 的地址访问一个 service 的时候，node2 上的负载均衡规则，就可能把这个 ip 包转发给一个在 node1 上的 pod。而当 node1 上的这个 pod 处理完请求之后，它就会按照这个 ip 包的源地址发出回复。如果没有 SNAT 操作的话，被转发来的 ip 包的源地址就是外部客户端的 ip 地址，pod 就会直接将回复发给客户端。对于客户端来说，它的请求明明发给了 node2，收到的回复却来自 node1，客户端很可能会报错。所以需要做 SNAT 操作，将源 ip 地址改成 node2 的 CNI 网桥地址或者 node2 自己的地址。这样，pod 在处理完成之后就会先回复给 node2，然后再由 node2 发送给客户端

不过，通过 SNAT 操作之后，接收请求的 pod 只知道该 ip 包来自于 node2，而不是外部的客户端。这时候，可以将 service 的 `spec.externalTrafficPolicy` 字段设置为 local，这样一台宿主机上的 iptables 规则，会设置为只将 ip 包转发给运行在这台宿主机上的 pod。这时，pod 就可以直接使用源地址将回复包发出，不需要事先进行 SNAT 操作了，并且保证了所有 pod 通过 service 收到请求之后，一定可以看到真正的、外部客户端的源地址。当然，这也就意味着如果在一台宿主机上，没有任何一个被代理的 pod 存在，如果使用该宿主机的 ip 地址访问 service，那么请求会被直接 drop 掉

### LoadBalancer
提供一个内部 ip，并在每个节点上启用一个端口来暴露服务。除此之外，Kubernetes 会请求底层云平台上的负载均衡器，将每个节点作为后端添加进去
```sh
kubectl expose pod app-pod --type=LoadBalancer
```

### ExternalName
```yaml
spec:
  type: ExternalName
  externalName: xx.database.xx.com
```
这时候访问 `xx-service.default.svc.cluster.local`，Kubernetes 返回的就是 `xx.database.example.com`。所以说，ExternalName 类型的 service，其实是在 kube-dns 里为你添加了一条 CNAME 记录。这时，访问 `xx-service.default.svc.cluster.local` 就和访问 `xx.database.example.com` 这个域名是一个效果


此外，Kubernetes 还允许为 service 分配公有 ip 地址
```yaml
externalIPs:
  - 80.11.12.10
```
可以通过 80.11.12.10 访问到被代理的 pod


## service 原理
### 实现方式
#### iptables
当 service 提交给 Kubernetes 之后，kube-proxy 就可以通过 service 的 Informer 感知到这样一个 service 对象的添加。作为对这个事件的响应，它会在宿主机上创建一条 iptables 规则:
凡是目的地址是 10.0.1.175、目的端口是 80 的 ip 包，都跳转到另外一条名叫 KUBE-SVC-XXX 的 iptables 链进行处理。其中 10.0.1.175 是这个 service 的 vip，所以这条 iptables 规则为这个 service 提供了固定的入口地址。而且由于 10.0.1.175 只是一条 iptables 规则上的配置，并没有真正的网络设备，所以你 ping 这个地址，是不会有任何响应的

即将跳转到的 KUBE-SVC-XXX 规则，是一组随机模式的 iptables 链。而随机转发的目的地，是三条链，这三条链最终目的地，就是这个 service 代理的三个 pod

查看 iptables 规则：
```sh
iptables-save
```

#### ipvs
基于 iptables 的 service 实现，是制约 Kubernetes 项目承载更多量级的 pod 的主要障碍。而 ipvs 模式的 service，能够有效的解决这个问题

在 ipvs 模式下，创建 service 之后，kube-proxy 会在宿主机上创建一个虚拟网卡（kube-ipvs0），并为它分配 vip。然后，kube-proxy 通过 Linux 的 ipvs 模块，为这个 ip 地址设置三个 ipvs 虚拟主机，并设置这三个虚拟主机之间使用轮询模式 (rr) 来作为负载均衡策略。相比于 iptables，ipvs 在内核中的实现其实也是基于 Netfilter 的 NAT 模式，所以在转发这一层上，理论上 ipvs 并没有显著的性能提升。但是，ipvs 并不需要在宿主机上为每个 pod 设置 iptables 规则，而是把对这些规则的处理放到了内核态，从而极大地降低了维护这些规则的代价

需要注意的是，ivps 模块只负责上述的负载均衡和代理功能。而一个完整的 service 流程正常工作所需要的包过滤、SNAT 等操作，还是要靠 iptables 来实现。只不过，这些辅助性的 iptables 规则数量有限，也不会随着 pod 数量的增加而增加

加载 ipvs 模块：
```sh
lsmod | grep ip_vs
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack_ipv4
```
修改为 ipvs 模式
```sh
kubectl edit configmap kube-proxy -n kube-system
mode: "ipvs"
```

重建 kube-proxy 生效配置：
```sh
kubectl delete pod kube-proxy-xxx -n kube-system
```

工具查看规则：
```sh
yum install ipvsadm -y
ipvsadm -L -n
```


## service spec
### ports
service.spec.ports

port: service port
nodePort: node port
targetPort: pod 的端口号或者端口名称


## service 网络问题
1. 无法通过 DNS 访问
检查 Kubernetes 自己的 Master 节点的 service DNS 是否正常
```sh
# 在一个 pod 里执行
nslookup kubernetes.default
```
如果访问 kubernetes.default 返回的值都有问题，那么需要检查 kube-dns 的运行状态和日志；反之，检查自己的 service 定义

2. 无法通过 ClusterIP 访问
首先应该检查的是这个 service 是否有 Endpoints
```sh
kubectl get endpoints <service name>
```
需要注意的是，如果你的 pod 的 readniessProbe 没通过，它也不会出现在 Endpoints 列表里；如果 Endpoints 正常，那么你就需要确认 kube-proxy 是否在正确运行；如果 kube-proxy 一切正常，就应该查看宿主机上的 iptables 了
