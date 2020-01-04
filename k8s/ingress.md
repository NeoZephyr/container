## Ingress
Ingress Controller 会根据定义好的 Ingress 对象，代理不同后端 Service 实现负载均衡

## Ingress Controller
```sh
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/mandatory.yaml
```

使用宿主机网络
```yaml
hostNetwork: true
```

Nginx Ingress Controller 提供一个可以根据 Ingress 对象和被代理后端 Service 的变化，来自动进行更新的 Nginx 负载均衡器

Nginx Ingress Controller 监听 Ingress 对象以及它所代理的后端 Service 变化。当一个新的 Ingress 对象由用户创建后，Nginx Ingress Controller 就会根据 Ingress 对象里定义的内容，生成一份对应的 Nginx 配置文件，并使用这个配置文件启动一个 Nginx 服务。而且一旦 Ingress 对象被更新，Nginx Ingress Controller 就会更新这个配置文件。需要注意的是，如果这里只是被代理的 Service 对象被更新，Nginx Ingress Controller 所管理的 Nginx 服务是不需要重新加载的

Nginx Ingress Controller 允许通过 Kubernetes 的 ConfigMap 对象来对上述 Nginx 配置文件进行定制。在这个 ConfigMap 里添加的字段，将会被合并到最后生成的 Nginx 配置文件当中。这个 ConfigMap 的名字，需要以参数的方式传递给 nginx-ingress-controller

Ingress Controller 允许在 pod 启动命令里设置 default-backend-service 参数，添加一条默认规则，例如 default-backend-service=nginx-default-backend。这样，任何匹配失败的请求，就都会被转发到这个名叫 nginx-default-backend 的 Service

### 高可用
如果域名只解析到一台 Ingress Controller，存在单点问题

#### 主备
通过 DaemonSet 结合 NodeSelector 在固定的两个个节点上启动 Ingress Controller，然后使用 keepalived 做主备，用户通过 vip 访问

#### 集群
前端 Load Balancer 将请求转发到集群中的某一个 Ingress Controller


## Ingress spec
ingress.spec.rules.host: 必须是一个标准的域名格式的字符串，而不能是 ip 地址。host 字段定义的值，就是这个 Ingress 的入口

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: helloworld-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  tls:
  - hosts:
    - www.helloworld.com
    secretName: helloworld-secret
  rules:
  - host: www.helloworld.com
    http:
      paths:
      - path: /customer
        backend:
          serviceName: customer-svc
          servicePort: 80
      - path: /member
        backend:
          serviceName: member-svc
          servicePort: 80
```

个性化配置
kubernetes.io/ingress.class: "nginx"
nginx.ingress.kubernetes.io/proxy-connect-timeout: "600" nginx.ingress.kubernetes.io/proxy-send-timeout: "600" nginx.ingress.kubernetes.io/proxy-read-timeout: "600" nginx.ingress.kubernetes.io/proxy-body-size: "10m"
nginx.ingress.kubernetes.io/ssl-redirect: "true"


### tls
```sh
cat > ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "kubernetes": {
         "expiry": "87600h",
         "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ]
      }
    }
  }
}
EOF
```
```sh
cat > ca-csr.json <<EOF
{
    "CN": "kubernetes",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "Beijing",
            "ST": "Beijing"
        }
    ]
}
EOF
```
```sh
cfssl gencert -initca ca-csr.json | cfssljson -bare ca -
```
```sh
cat > www.clink.com-csr.json <<EOF
{
  "CN": "www.clink.com",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "BeiJing",
      "ST": "BeiJing"
    }
  ]
}
EOF
```
```sh
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes www.clink.com-csr.json | cfssljson -bare www.clink.com 

kubectl create secret tls www-clink-comm --cert=www.clink.com.pem --key=www.clink.com-key.pem
```