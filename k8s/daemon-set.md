## DaemonSet 的作用
1. 保证集群内每一个（或一些）节点都运行一组相同的 pod
2. 跟踪集群节点状态，保证新加入的节点自动创建对应的 pod
3. 跟踪集群节点状态，保证移除的节点删除对应的 pod
4. 跟踪 pod 状态，保证每个节点 pod 处于运行状态

## DaemonSet 原理
在创建每个 pod 的时候，DaemonSet 会自动给这个 pod 加上一个 nodeAffinity，从而保证这个 pod 只会在指定节点上启动。同时，它还会自动给这个 pod 加上一个 Toleration，从而忽略节点的 unschedulable 污点

DaemonSet 控制器直接操作 pod，不需要像 deployment 那样通过 ReplicaSet 控制版本