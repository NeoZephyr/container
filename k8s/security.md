## 安全框架
1. 访问集群的资源需要过鉴权、鉴权、准入控制
2. 普通用户若要安全访问 api-server，需要证书、token 或者通过用户名加密码得分方式访问
3. pod 访问 api-server 需要 ServiceAccount
4. 基于 https 的传输方式保证传输安全


## 鉴权
1. 基于 ca 证书签名的数字证书认证
2. 通过 token 来识别用户
3. 用户名加密码的方式认证

生成证书
```sh
cat > ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "kubernetes": {
        "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ],
        "expiry": "87600h"
      }
    }
  }
}
EOF
```
```sh
cat > clink-csr.json <<EOF
{
  "CN": "clink",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
EOF
```

```sh
wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
chmod +x cfssl*
mv cfssl_linux-amd64 /usr/bin/cfssl
mv cfssljson_linux-amd64 /usr/bin/cfssljson
mv cfssl-certinfo_linux-amd64 /usr/bin/cfssl-certinfo
```

```sh
cfssl gencert -ca=/etc/kubernetes/pki/ca.crt -ca-key=/etc/kubernetes/pki/ca.key -config=ca-config.json -profile=kubernetes clink-csr.json | cfssljson -bare clink
```

生成 kubeconfig 授权文件
```sh
kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/pki/ca.crt \
  --embed-certs=true \
  --server=https://192.168.20.11:6443 \
  --kubeconfig=clink.kubeconfig

# 设置客户端认证
kubectl config set-credentials clink \
  --client-key=clink-key.pem \
  --client-certificate=clink.pem \
  --embed-certs=true \
  --kubeconfig=clink.kubeconfig

# 查看上下文
kubectl config get-contexts

# 设置默认上下文
kubectl config set-context kubernetes \
  --cluster=kubernetes \
  --user=clink \
  --kubeconfig=clink.kubeconfig

# 设置当前使用配置
kubectl config use-context kubernetes --kubeconfig=clink.kubeconfig
# kubectl config use-context kubeadm
# kubectl config delete-context kubeadm
```

```sh
kubectl --kubeconfig=clink.kubeconfig get pods
kubectl --kubeconfig=clink.kubeconfig get pods -n kube-system

mv clink.kubeconfig .kube/config
```

## 授权
RBAC：基于角色的访问控制，负责完成授权工作

### 角色
1. Role：定义了一组对特定命名空间 Kubernetes API 对象的操作权限
2. ClusterRole：定义了一组对所有命名空间 Kubernetes API 对象的操作权限

一个 Kubernetes API 对象在 Etcd 里的完整资源路径，是由 API Group、API Version 和 API Resource 三个部分组成的

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: read-pod
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: read-pod-cluster
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
```

可以只针对某一个具体的对象进行权限设置
```yaml
rules:
- apiGroups: [""]
  resources: ["pods"]
  resourceNames: ["nginx"]
  verbs: ["get"]
```

在 Kubernetes 中已经内置了很多个为系统保留的 ClusterRole，它们的名字都以 system: 开头
```sh
kubectl get clusterroles
kubectl describe clusterrole system:kube-scheduler
```

Kubernetes 还提供了四个预先定义好的 ClusterRole 来供用户直接使用
1. cluster-admin：整个 Kubernetes 项目中的最高权限
2. admin
3. edit
4. view：被作用者只有 Kubernetes API 的只读权限

### 主体
1. User：用户
2. Group：用户组
3. ServiceAccount：服务账号

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  namespace: default
  name: read-pod
```

一个 ServiceAccount，在 Kubernetes 里对应的用户的名字是：
```
system:serviceaccount:<ServiceAccount 名字>
```

对应的内置用户组的名字是：
```
system:serviceaccounts:<Namespace 名字>
```

Role 的权限规则，作用于 default 里的所有 ServiceAccount
```yaml
subjects:
- kind: Group
  name: system:serviceaccounts:default
  apiGroup: rbac.authorization.k8s.io
```

Role 的权限规则，作用于整个系统里的所有 ServiceAccount
```yaml
subjects:
- kind: Group
  name: system:serviceaccounts
  apiGroup: rbac.authorization.k8s.io
```

### 角色绑定
1. RoleBinding：角色到主体的绑定关系
2. ClusterRoleBinding：集群角色到主体的绑定关系

在 Kubernetes 项目中，负责完成授权（Authorization）工作的机制，就是 RBAC

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pod
  namespace: default
subjects:
- kind: User
  name: clink
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: read-pod
  apiGroup: rbac.authorization.k8s.io
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: read-pod-cluster
subjects:
- kind: User
  name: clink
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: read-pod-cluster
  apiGroup: rbac.authorization.k8s.io
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pod-sa
subjects:
- kind: ServiceAccount
  name: read-pod
roleRef:
  kind: Role
  name: read-reader
  apiGroup: rbac.authorization.k8s.io
```


## 准入控制
发送到 api-server 的请求都需要经过准入控制器插件列表，进行检查


## ServiceAccount
Kubernetes 会为一个 ServiceAccount 自动创建并分配一个 Secret 对象（ServiceAccountToken），ServiceAccount 的授权信息和文件都保存在这个 Secret 对象里面

```sh
kubectl get sa default -o yaml
```

### ServiceAccount 原理
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-with-sa
spec:
  containers:
  - name: nginx
    image: nginx
  serviceAccountName: nginx-sa
```

pod 运行起来之后，ServiceAccount 的 token，也就是 Secret 对象，被 Kubernetes 自动挂载到了容器的 `/var/run/secrets/kubernetes.io/serviceaccount` 目录下

```sh
kubectl describe pod nginx-with-sa
```

应用程序只需要加载这些授权文件（ca.crt 等），就可以访问并操作 Kubernetes API 了
```sh
kubectl exec -it nginx-with-sa -- /bin/bash
ls /var/run/secrets/kubernetes.io/serviceaccount
```

### 默认 ServiceAccount
如果一个 pod 没有声明 serviceAccountName，Kubernetes 会自动在它的 Namespace 下创建一个名叫 default 的默认 ServiceAccount，然后分配给这个 pod。但此时，这个默认的 ServiceAccount 并没有关联任何 Role，具有访问 api-server 的绝大多数权限。在生产环境中，建议为所有 Namespace 下的默认 ServiceAccount，绑定一个只读权限的 Role
