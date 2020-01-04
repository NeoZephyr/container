## ConfigMap
管理容器运行所需的配置文件、环境变量、命令行参数等可变配置，用于解耦容器镜像和可变配置

### 创建 ConfigMap
手动命令创建
```sh
kubectl create configmap app-config --from-file=app.yaml
kubectl create configmap global-config --from-literal=thread-num=10 --from-literal=consumer-num=20
```

yaml 文件创建
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: user-config
  namespace: default
data:
  username: "admin"
  password: "654321"
```
```yaml
# 多行
apiVersion: v1
kind: ConfigMap
metadata:
  name: redis-config
data:
  redis.properties: |
    redis.host=127.0.0.1
    redis.port=6379
    redis.password=123456
```

### 使用 ConfigMap
变量注入
```yaml
env:
- name: consumerNum
  valueFrom:
    configMapKeyRef:
      name: global-config
      key: consumer-num
- name: threadNum
  valueFrom:
    configMapKeyRef:
      name: global-config
      key: thread-num
```

挂载
```yaml
containers:
- name: busybox
  image: busybox
  volumeMounts:
  - name: redis-volume
    mountPath: /etc/redis
volumes:
- name: redis-volume
  configMap:
    name: redis-config
```
