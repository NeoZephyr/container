## emptyDir
创建一个空卷，挂载到 pod 中的容器，pod 删除该卷也会被删除。主要应用于 pod 中容器之间数据共享
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: empty-volume-pod
spec:
  containers:
  - name: producer
    image: centos
    command: ["bash", "-c", "for i in {1..1000};do echo $i >> /data/number;sleep 1;done"]
    volumeMounts:
    - name: data
      mountPath: /data

  - name: consumer
    image: centos
    command: ["bash", "-c", "tail -f /data/number"]
    volumeMounts:
    - name: data
      mountPath: /data

  volumes:
  - name: data
    emptyDir: {}
```


## hostPath
挂载节点文件系统上文件或者目录到 pod 中的容器，主要应用于 pod 中容器需要访问宿主机文件
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: host-path-pod
spec:
  containers:
  - name: consumer
    image: busybox
    args:
    - /bin/sh
    - -c
    - sleep 36000
    volumeMounts:
    - name: data
      mountPath: /opt/logs
  volumes:
  - name: data
    hostPath:
      path: /opt/logs
      type: DirectoryOrCreate
```


## 网络
### NFS
#### server
```sh
yum install nfs-utils -y
```
```sh
# /etc/exports
/opt/nfs *(rw,no_root_squash)
```
```sh
sudo systemctl start nfs
sudo systemctl enable nfs-utils
```
```sh
# 检查是否可以挂载
mount -t nfs <host>:/opt/nfs /mnt/
```

#### client
```sh
yum install nfs-utils -y

sudo systemctl start nfs
sudo systemctl enable nfs-utils
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: net-volume
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 3
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        volumeMounts:
        - name: webroot
          mountPath: /usr/share/nginx/html
        ports:
        - containerPort: 80
      volumes:
      - name: webroot
        nfs:
          server: 192.168.20.11
          path: /opt/nfs
```

```sh
curl <pod_ip>/index.html
```


## pvc
### pvc 的作用
1. 职责分离，pvc 中只需要声明需要存储的大小，访问模式等真正关心的存储需求。pv 和其对应的后端存储信息则交给 `cluster admin` 统一运维和管控，安全访问策略更容易控制
2. pvc 简化了用户对存储的需求，pv 才是存储的实际信息的承载体，通过 Controller Manager 中的 PersistentVolumeController 将 pvc 与合适的 pv 绑定到一起，从而满足用户对存储的实际需求


## pv 分类
pv 是对存储资源创建和使用的抽象，使得存储作为集群中的资源管理

### 静态 pv
需要管理员提前创建好 pv 以供使用。由于用户的需求是多样化的，静态 pv 很容易导致用户提交的 pvc 找不到合适的 pv

```sh
yum install nfs-utils
mkdir -p /opt/nfs
```
```sh
# vi /etc/exports
/opt/nfs *(rw,no_root_squash)
```
```sh
sudo systemctl start nfs
sudo systemctl enable nfs
```

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-small
spec:
  capacity:
    storage: 5Gi
  accessModes:
  - ReadWriteMany
  nfs:
    path: /opt/nfs/pv-small
    server: 192.168.20.11
```

使用 pv
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nginx-pvc
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 8Gi
```
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pv
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
    volumeMounts:
    - name: webroot
      mountPath: /usr/share/nginx/html
  volumes:
  - name: webroot
    persistentVolumeClaim:
      claimName: nginx-pvc
```

### 动态 pv
StorageClass 作为创建 pv 的模板，包含了创建某种具体类型 pv 所需的参数信息，用户只需要在 pvc 中指定使用的存储模板以及自己需要的大小、访问方式等参数，然后 k8s 结合 pvc 和 StorageClass 两者的信息动态创建 pv 对象

#### nfs
kubernetes-incubator/external-storage
```yaml
kubectl create -f class.yaml
kubectl create -f rbac.yaml
kubectl create -f deployment.yaml
```

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nginx-dynamic-pvc
spec:
  storageClassName: "managed-nfs-storage"
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 12Gi
```
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-dynamic-pv
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
    volumeMounts:
    - name: webroot
      mountPath: /usr/share/nginx/html
  volumes:
  - name: webroot
    persistentVolumeClaim:
      claimName: nginx-dynamic-pvc
```


## pv 原理
### pv 持久化过程
1. pod 调度到一个节点上，kubelet 为这个 pod 创建它的 Volume 目录
```
/var/lib/kubelet/pods/<PodID>/volumes/kubernetes.io~<VolumeType>/<VolumeName>
```
2. 虚拟机挂载远程磁盘，这个阶段称为 attach
3. 将磁盘设备格式化并挂载到 Volume 宿主机目录，这个阶段称为 Mount

相对应的，在删除一个 pv 的时候，Kubernetes 也需要 Unmount 和 Dettach 阶段

### pv 状态
1. pending
2. available
3. bound
4. released
5. failed
6. deleted

用户创建的 pvc 要真正被容器使用起来，必须找到符合以下条件的 pv
1. 满足 pvc 的 spec 字段中的要求，如存储大小等
2. pv 与 pvc 的 storageClassName 字段必须一样

如果在创建 pod 的时候，系统中没有符合条件的 pv 与它定义的 pvc 绑定，pod 启动就会报错。在 Kubernetes 中，Volume Controller 专门处理持久化存储，控制着多个循环。其中一个叫 PersistentVolumeController 的控制器，会不断查看当前每一个 pvc 是否处于 Bound 状态，如果不处于 Bound 状态，就会遍历所有可用的 pv，尝试与这个不在 Bound 状态的 pvc 进行绑定。pv 与 pvc 进行绑定，其实就是将这个 pv 对象的名字，填在了 pvc 对象的 spec.volumeName 字段上

达到 released 状态的 pv 无法根据 reclaim policy 回到 available 状态再次 bound 到新的 pvc。此时，要想复用原来的 pv 对应的存储中的数据，有以下两种方式：
1. 复用原 pv 中记录的存储信息新建 pv 对象
2. 直接从 pvc 对象复用，即不 unbound pvc 和 pv

在删除 pv 时需要按如下流程执行操作
1. 删除使用这个 pv 的 pod
2. 从宿主机移除本地磁盘
3. 删除 pvc
4. 删除 pv
