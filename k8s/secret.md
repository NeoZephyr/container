## Secret
Secret 加密数据并存放 Etcd 中

### 创建 Secret
手动命令创建
```sh
kubectl create secret generic user-secret --from-file=username.txt
kubectl create secret generic password-secret --from-literal=password=123456
```
```sh
kubectl create secret tls tls-demo-secret --cert=certfile --key=keyfile
```

yaml 文件创建
Secret 对象要求数据必须是经过 Base64 转码的，以免出现明文密码的安全隐患
```sh
echo -n 'admin' | base64
echo -n '123456' | base64
```
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: dbsecret
type: Opaque
data:
  username: YWRtaW4=
  password: MTIzNDU2
```

### 使用 Secret
变量注入
```yaml
env:
- name: username
  valueFrom:
    secretKeyRef:
      name: conf-secret
      key: username
- name: password
  valueFrom:
    secretKeyRef:
      name: conf-secret
      key: password
```

挂载
```yaml
containers:
- name: echo-file
  image: busybox
  volumeMounts:
  - name: user
    mountPath: "/etc/user"
    readOnly: true
volumes:
- name: user
  secret:
    secretName: conf-secret
```

```yaml
containers:
- name: echo-file
  image: busybox
  volumeMounts:
  - name: user
    mountPath: "/etc/user"
    readOnly: true
volumes:
- name: user
  projected:
    sources:
    - secret:
        name: conf-secret
```
