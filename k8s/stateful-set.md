## StatefulSet
StatefulSet 把真实世界里的应用状态抽象为两种状态：
1. 拓扑状态
2. 存储状态

StatefulSet 的核心功能，就是通过某种方式记录这些状态，然后在 pod 被重新创建时，能够为新 pod 恢复这些状态


## 拓扑状态
有状态应用是不能像无状态应用那样，创建一个标准 service，然后访问 ClusterIP 负载均衡到一组 pod 上，而是采用无头服务

```yaml
apiVersion: v1
kind: Service
metadata:
  name: headless-nginx-service
spec:
  clusterIP: None
  selector:
    app: headless-nginx
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: headless-nginx
spec:
  selector:
    matchLabels:
      app: headless-nginx
  serviceName: headless-nginx-service
  replicas: 3
  template:
    metadata:
      labels:
        app: headless-nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
          name: nginx
```

StatefulSet 为管理的所有 pod 的名字进行了编号，并且按照编号顺序逐一完成创建工作。如果将所有 pod 删除，StatefulSet 会保证按照原来的编号顺序创造出两个新的 pod。通过这种方式，pod 的拓扑状态，即节点启动的先后顺序按照名字 + 编号的方式固定了下来

获取无头服务代理的所有 pod 的 ip 地址和 pod 的 DNS A 记录
```sh
nslookup headless-nginx-service.default.svc.cluster.local
```

通过 pod 的 DNS 名称获取具体的 ip 地址，DNS 名称作为 pod 的固定身份，在生命周期不会再变化
```sh
nslookup headless-nginx-0.headless-nginx-service.default.svc.cluster.local
```

对于有状态应用实例的访问，必须使用 DNS 记录或者 hostname 的方式，而不是直接访问这些 pod 的 ip 地址


## 存储状态
StatefulSet 会根据 volumeClaimTemplates 为每个 pod 声明一个带编号的 pvc。自动创建的 pvc 与 pv 绑定成功后，就会进入 Bound 状态。这样，就可以确保每一个 pod 都拥有一个独立的 Volume

删除 pod 或 StatefulSet 时，它所对应的 pvc 和 pv 不会被删除。因此，当这个 pod 被重新创建之后，Kubernetes 会为它找到同样编号的 pvc，挂载这个 pvc 对应的 Volume，从而获取到以前保存在 Volume 里的数据

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: headless-volume-nginx
spec:
  selector:
    matchLabels:
      app: nginx 
  serviceName: "headless-volume-nginx"
  replicas: 3 
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx 
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: webroot
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: webroot
    spec:
      accessModes:
      - ReadWriteOnce
      storageClassName: "managed-nfs-storage"
      resources:
        requests:
          storage: 1Gi
```