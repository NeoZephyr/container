## master
### api-server
负责 API 服务的

### controller-manager
负责管理控制器

### scheduler
根据调度算法为新创建的 pod 选择一个 Node 节点

### etcd
整个集群的持久化数据，由 api-server 处理后保存在 Ectd 中

## worker
### kubelet
管理本机运行容器的生命周期，比如创建容器、pod 挂载数据卷、下载 secret、获取容器和节点状态 等工作。kubelet 将每个 pod 转换成一组容器

### kube-proxy

### docker
容器引擎


