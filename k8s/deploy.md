## deploy 作用
1. 保证集群内可用 pod 的数量
2. 方便为所有 pod 更新镜像版本
3. 更新过程中，保证服务可用性
4. 更新过程中，发现问题，快速回滚


## deploy 操作
```sh
kubectl create deployment app --image=nginx

# --record 表示记录每次操作所执行的命令
kubectl create -f app-deployment.yaml --record

kubectl delete deployment app
kubectl get deployment
kubectl get deployment -o wide
```

```sh
kubectl set image deploy <deploy name> <container name>=<new image>
```

```sh
kubectl rollout history deploy <deploy name>

# 查看具体版本的细节
kubectl rollout history deploy <deploy name> --revision=2

kubectl rollout undo deploy <deploy name>
kubectl rollout undo deploy <deploy name> --to-revision=2
kubectl rollout status deploy <deploy name>
```

暂停，可以随意使用 `kubectl edit` 或者 `kubectl set image` 指令而不会触发滚动更新，也不会创建新的 ReplicaSet
```
kubectl rollout pause deploy <deploy name>
```

恢复，在 pause 与 resume 之间的这段时间对 Deployment 的修改只会触发一次滚动更新
```sh
kubectl rollout resume deploy <deploy name>
```

deploy.spec.revisionHistoryLimit
Kubernetes 为 Deployment 保留的历史版本个数，如果把它设置为 0，就不能进行回滚操作了


## deploy 原理
### ReplicaSet
deployment 负责管理不同版本的 replicaSet，每个 replicaSet 对应 deployment template 的一个版本。replicaSet 管理 pod，一个 replicaSet 下的 pod 都是相同版本的

```sh
kubectl get replicaSet
```
```sh
# 水平扩展/收缩
kubectl scale deployment <deploy name> --replicas=4
```

### 状态
DESIRED：用户期望的 pod 副本个数
CURRENT：当前处于 Running 状态的 pod 的个数
UP-TO-DATE：当前处于最新版本的 pod 的个数
AVAILABLE：当前已经可用的 pod 的个数，即：既是 Running 状态，又是最新版本，并且已经处于 Ready（健康检查正确）状态的 pod 的个数

### 控制循环
不断使系统向最终态趋近 spec 中声明的状态

sensor
1. Reflector 通过 list（全量）和 watch（增量）获取 api-server 数据
2. Reflector 获取新的资源数据后，会在 delta 队列中加入一条包括资源对象信息、资源对象事件类型的记录
3. delta 保证同一个对象中仅有一条记录
4. Informer 不断从 delta 队列中获取记录，将记录交给记录的回调函数，同时将资源对象交给 Indexer，让 Indexer 把资源对象记录在缓存中

controller
1. 事件处理函数监听 Informer 中资源新增、更新、删除事件，并根据控制器逻辑决定是否需要处理。对需要处理的事件，会把事件关联资源的命名空间以及名字加入工作队列中。工作队列会对存储对象进行去重
2. woker 从队列中取出信息，并根据资源名字获取最新资源数据，用来创建、更新资源对象，或者调用外部服务
3. worker 处理失败时，会将资源名字重新加入队列中以进行重试