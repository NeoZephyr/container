## 容器关系
### 亲密关系
调度解决，将两个应用调度到同一个节点

### 超亲密关系
1. 发生直接的文件交换
2. 使用 localhost 或者 socket 文件进行本地通信
3. 频繁的 rpc 调用
4. 共享某些 Linux Namespace

假设有 4 个联系紧密的进程运行在容器中，除了 pid = 1 的进程，剩下的 3 个进程无法被管理，因为容器是单进程模型。有以下两种方式解决该问题：
1. 应用程序本身具备进程管理能力（systemd）
2. 将容器 pid = 1 的进程改为 systemd，这会导致管理容器 = 管理 systemd，而不是直接管理应用本身


## pod 的作用
1. pod 作为 k8s 中最小调度单位
2. pod 将联系紧密的容器组合起来，便于内部协作，避免了这些容器单元被调度到不同的节点上
3. pod 内的容器共享网络命名空间等资源，并且可以声明共享同一个 Volume
4. kubernetes = 操作系统，容器 = 进程，pod = 进程组


## pod 原理
### Infra Container
其他容器通过加入基础容器的方式共享同一个 Network Namespace，pod 内容器共用一个 ip 地址，也就是这个 pod 的 Network Namespace 对应的 ip 地址。因此，容器之间可以直接通过 localhost 进行通信。pod 的生命周期与 Infra 容器一致，与其它容器无关

### InitContainers
1. 优先于普通容器启动执行，当所有 InitContainer 执行成功之后，普通容器才被启动
2. pod 中多个 InitContainer 之间是按照次序依次启动执行，pod 中的多个普通容器是并行启动，而且需要等到 InitContainers 都启动并且退出了，用户容器才会启动
3. InitContainer 执行成功后就结束退出了，而普通容器可能会一直执行或者重启
4. 一般 InitContainer 用于普通容器启动前的初始化或者前置条件检验

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  initContainers:
  - image: pain/app:1.1.0
    name: war
    command: ["cp", "/app.war", "/app"]
    volumeMounts:
    - mountPath: /app
      name: app-volume
  containers:
  - image: pain/tomcat:8.5
    name: tomcat
    command: ["sh", "-c", "/root/apache-tomcat-7/bin/start.sh"]
    volumeMounts:
    - mountPath: /root/apache-tomcat-7/webapps
      name: app-volume
    ports:
    - containerPort: 8080
      hostPort: 8001
  volumes:
  - name: app-volume
    emptyDir: {}
```

### Containers
并行启动

Sidecar
通过在 pod 里面定义专门容器，来执行主业务需要的辅助工作，比如
1. 需要 ssh 进去执行的脚本
2. 日志收集
3. debug 应用
4. 应用监控


## pod spec
### imagePullPolicy
pod.spec.containers.imagePullPolicy
镜像拉取策略

1. IfNotPresent
2. Always
3. Never

### imagePullSecrets
pod.spec.imagePullSecrets

### 资源限制
spec.containers[].resources.limits.cpu
spec.containers[].resources.requests.cput
spec.containers[].resources.limits.memory
spec.containers[].resources.requests.memory
spec.containers[].resources.limits.ephemeral-storage
spec.containers[].resources.requests.ephemeral-storage

CPU 资源属于可压缩资源，当资源不足时，pod 只会饥饿，但不会退出；内存资源属于不可压缩资源，当资源不足时，pod 就会因为 OOM（Out-Of-Memory）被内核杀掉。由于 pod 由多个 container 组成，因此 pod 整体的资源配置，就由这些 container 的配置累加得到

Kubernetes 在调度的时候，scheduler 按照 requests 的值进行计算。而在真正设置 cgroups 限制的时候，kubelet 则会按照 limits 的值来进行设置

pod 服务质量分类:
1. 当 pod 里的每一个 container 都同时设置了 requests 和 limits，并且 requests 和 limits 值相等的时候，这个 pod 就属于 Guaranteed 类别
2. 当 pod 仅设置了 limits 没有设置 requests 的时候，Kubernetes 会自动为它设置与 limits 相同的 requests 值，也属于 Guaranteed 类别
3. 当 pod 不满足 Guaranteed 的条件，但至少有一个 container 设置了 requests。那么这个 pod 就会被划分到 Burstable 类别
4. 而如果一个 pod 既没有设置 requests，也没有设置 limits，那么它的 QoS 类别就是 BestEffort

当节点上不可压缩资源不足时，就有可能触发 Eviction。比如，可用内存（memory.available）、可用的宿主机磁盘空间（nodefs.available），以及容器运行时镜像存储空间（imagefs.available）等

Eviction 在 Kubernetes 里分为 Soft 和 Hard 两种模式，Soft Eviction 允许你为 Eviction 过程设置一段优雅时间，例如设置 imagefs.available=2m 意味着当 imagefs 不足的阈值达到 2 分钟之后，kubelet 才会开始 Eviction；而 Hard Eviction 模式下，Eviction 过程就会在阈值达到之后立刻开始

当宿主机的 Eviction 阈值达到后，就会进入 MemoryPressure 或者 DiskPressure 状态，从而避免新的 Pod 被调度到这台宿主机上

当 Eviction 发生时，kubelet 会参考 pod 的 QoS 类别进行删除操作
1. 首先驱逐 BestEffort 类别的 pod
2. 其次，是属于 Burstable 类别、并且发生饥饿的资源使用量已经超出了 requests 的 pod
3. 最后，才是 Guaranteed 类别。并且，Kubernetes 会保证只有当 Guaranteed 类别的 pod 的资源使用量超过了其 limits 的限制，或者宿主机本身正处于 Memory Pressure 状态时，Guaranteed 的 pod 才可能被选中进行 Eviction 操作
4. 对于同 QoS 类别的 pod 来说，Kubernetes 还会根据 pod 的优先级来进行进一步地排序和选择

通过设置 cpuset 把容器绑定到某个 CPU 的核上，这样，由于操作系统在 CPU 之间进行上下文切换的次数大大减少，容器里应用的性能会得到大幅提升。不过需要满足以下条件：
1. pod 必须是 Guaranteed 的 QoS 类型
2. 将 pod 的 CPU 资源的 requests 和 limits 设置为同一个相等的整数值
```yaml
spec:
  containers:
  - name: nginx
    image: nginx
    resources:
      limits:
        memory: "200Mi"
        cpu: "2"
      requests:
        memory: "200Mi"
        cpu: "2"
```
这时候，该 Pod 就会被绑定在 2 个独占的 CPU 核上

建议将 DaemonSet 的 pod 都设置为 Guaranteed 的 QoS 类型

### restartPolicy
pod.spec.restartPolicy

1. Always 默认策略，总是重启
2. OnFailure，失败才重启
3. Never，永远不重启

对于包含多个容器的 pod，只有它里面所有的容器都进入异常状态后，pod 才会进入 Failed 状态

### hostAliases
pod.spec.hostAliases

定义 pod 的 hosts 文件（比如 /etc/hosts）里的内容
```yaml
apiVersion: v1
kind: Pod
...
spec:
  hostAliases:
  - ip: "10.1.2.3"
    hostnames:
    - "foo.remote"
    - "bar.remote"
...
```

### 生命周期
pod.spec.containers.lifecycle

在容器状态发生变化时触发一系列钩子
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: lifecycle-demo
spec:
  containers:
  - name: lifecycle-demo-container
    image: nginx
    lifecycle:
      postStart:
        exec:
          command: ["/bin/sh", "-c", "echo Hello from the postStart handler > /usr/share/message"]
      preStop:
        exec:
          command: ["/usr/sbin/nginx", "-s", "quit"]
```

1. Pending：YAML 文件已经提交给了 Kubernetes，API 对象已经被创建并保存在 Etcd 当中。但是，这个 pod 里有些容器因为某种原因而不能被顺利创建。比如，调度不成功
2. Running：pod 已经调度成功，跟一个具体的节点绑定。它包含的容器都已经创建成功，并且至少有一个正在运行中
3. Succeeded：pod 里的所有容器都正常运行完毕，并且已经退出了。这种情况在运行一次性任务时最为常见
4. Failed：pod 里至少有一个容器以不正常的状态（非 0 的返回码）退出
5. Unknown：pod 的状态不能持续地被 kubelet 汇报给 api-server，这很有可能是主从节点间的通信出现了问题


### 健康检查
pod.spec.containers.livenessProbe
pod.spec.containers.readinessProbe

#### 探测方式
1. httpGet: 返回 200-399 状态码表明容器健康
2. exec: 通过执行命令进行检查，命令返回为 0 表示容器健康
3. tcpSocket: 通过容器 ip 和 port 执行 tcp 检查，如果能建立 tcp 连接则表明容器健康

```yaml
livenessProbe:
  exec:
    command:
    - cat
    - /tmp/healthy
  initialDelaySeconds: 5
  periodSeconds: 5
```
```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
    httpHeaders:
    - name: X-Custom-Header
      value: Awesome
    initialDelaySeconds: 3
    periodSeconds: 3
```

#### 探测结果
1. success 通过了检查
2. failure 未通过检查
3. unknown 未能执行检查，不采取任何动作

#### 检测类型
1. liveness 检查失败，杀死 pod，然后根据 restartPolicy 来操作
2. readness 检查失败，将 pod 从 service endpoints 中移出，切断上层流量到 pod（将 pod 从 endpoint 中移除）

#### 探测参数
1. initialDelaySeconds: pod 启动后延迟多久进行检查
2. periodSeconds: 检查的间隔时间，默认 10s
3. timeoutSeconds: 探测的超时时间，默认 1s
4. successThreshold: 探测失败后再次判断成功的阈值次数，默认为 1
5. failureThreshold: 探测确定为失败的需要的次数，默认为 3

选择合适的探测方式
1. 调大判断的超时阈值，防止在容器压力较高的情况下出现偶发超时
2. 调整判断的次数阈值，默认的 3 次在短周期下不一定是最佳实践
3. exec 如果执行的是 shell 脚本，在容器中可能调用时间会非常长


## pod 操作
```sh
kubectl create deploy nginx --image=nginx:1.14 -o yaml --dry-run > nginx-deploy.yaml

kubectl get pods <pod name> -o yaml --export > nginx-pod.yaml
```
```sh
kubectl create -f nginx_pod.yaml
kubectl delete -f nginx_pod.yaml
```
```sh
kubectl get pods --show-labels
kubectl get pods -l app
kubectl get pods -l app=nginx

kubectl label pods <pod name> <labelName>=<labelValue>
kubectl label pods <pod name> <labelName>=<labelValue> --overwrite

kubectl get pods -o wide
kubectl describe pods <pod name>
```
```sh
kubectl exec -it <pod name> /bin/bash
kubectl port-forward <pod name> 8080:80
```

调试工具
telepresence
kubectl debug


## pod 调度
### 调度流程
1. 调用 api-server 创建 pod，将状态写入到 etcd
2. scheduler 监听 api-server 获知新 pod 创建，通过调用 api-server 将 pod 绑定到合适的节点，并将状态写入到 etcd
3. kubelet 监听 api-server 获知新 pod 绑定到节点上，调用 docker api 创建容器。然后，通过调用 api-server 更新 pod 状态，并写入到 etcd

### 调度策略
1. pod.spec.nodeName
用于将 pod 调度到指定名称的节点上。一旦 pod 的这个字段被赋值，Kubernetes 就会认为这个 pod 已经经过了调度，调度的结果就是赋值的节点名字

2. pod.spec.nodeSelector
用于将 pod 调度到匹配标签的节点上

3. pod.spec.tolerations
```
kubectl taint nodes <node name> <key>=<value>:[effect]
kubectl describe nodes <node name>

kubectl taint nodes <key>-
```
一旦某个节点被加上了一个 Taint，那么所有 pod 就都不能在这个节点上运行。pod 需要声明自己能容忍这个污点，才可以在这个节点上运行

effect 有如下取值
NoSchedule: 这个 taint 只会在调度新 pod 时产生作用
PreferNoSchedule: 尽量不要调度
NoExecute: 不仅不会调度，还会驱逐 node 上已有的 pod

```yaml
apiVersion: v1
kind: Pod
...
spec:
  tolerations:
  - key: "foo"
    operator: "Equal"
    value: "bar"
    effect: "NoSchedule"
```

容忍所有以键为 foo 的 Taint
```yaml
apiVersion: v1
kind: Pod
...
spec:
  tolerations:
  - key: "foo"
    operator: "Exists"
    effect: "NoSchedule"
```