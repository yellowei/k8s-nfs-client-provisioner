
## 为集群配置动态分配存储空间的StorageClass

### 配置NFS服务

在此之前，我们要先为Ubuntu系统设置NFS（一种存储共享方式，可自行百度），我们一般选择一台worker节点部署NFS Server就够用了

#### 1. 安装nfs服务端

```shell
sudo apt install nfs-kernel-server -y
```
 
#### 2. 创建目录，用于作为共享卷的根目录，后续k8s的pod通过pvc申请到的pv会在这里

```shell
sudo mkdir -p /data/k8s/
```
 
#### 3. 使任何客户端均可访问

```shell
sudo chown nobody:nogroup /data/k8s/　　
#sudo chmod 755 /data/k8s/
sudo chmod 777 /data/k8s/
```
 
#### 4. 配置/etc/exports文件, 使任何ip均可访问(加入以下语句)

```shell
vi /etc/exports
/data/k8s/ *(rw,sync,no_subtree_check)
```
　　
#### 5. 检查nfs服务的目录

sudo exportfs -ra (重新加载配置)
sudo showmount -e (查看共享的目录和允许访问的ip段)

```shell
yellowei@k8s-worker-node1:/data/k8s$ sudo showmount -e
Export list for k8s-worker-node1:
/data/k8s *
```

 
#### 6. 重启nfs服务使以上配置生效

```shell
sudo systemctl restart nfs-kernel-server
#sudo /etc/init.d/nfs-kernel-server restart
```

查看nfs服务的状态是否为active状态:active(exited)或active(runing)

```shell
yellowei@k8s-worker-node1:/data/k8s$ systemctl status nfs-kernel-server
● nfs-server.service - NFS server and services
     Loaded: loaded (/lib/systemd/system/nfs-server.service; enabled; vendor preset: enabled)
     Active: active (exited) since Sun 2022-03-06 23:15:08 CST; 2h 21min ago
    Process: 34974 ExecStartPre=/usr/sbin/exportfs -r (code=exited, status=0/SUCCESS)
    Process: 34975 ExecStart=/usr/sbin/rpc.nfsd $RPCNFSDARGS (code=exited, status=0/SUCCESS)
   Main PID: 34975 (code=exited, status=0/SUCCESS)

Mar 06 23:15:07 k8s-worker-node1 systemd[1]: Starting NFS server and services...
Mar 06 23:15:08 k8s-worker-node1 systemd[1]: Finished NFS server and services.
```
 
#### 7. 测试nfs服务是否成功启动
 
回到Master节点，在Master节点安装nfs 客户端

```shell
sudo apt-get install nfs-common
```

创建挂载目录

```shell
 sudo mkdir /data/k8s/
```
 
将worker节点上的nfs共享卷挂载到master节点的/data/k8s/
```shell
sudo mount -t nfs -o nolock -o tcp 192.168.223.131:/data/k8s/ /data/k8s/(挂载成功，说明nfs服务正常)
```

### 8. 在所有节点上安装nfs-common

如果发现部署pod报错如下

```shell
MountVolume.SetUp failed for volume "nfs-client-root" : mount failed: exit status 32 Mounting command: mount Mounting arguments: -t nfs 192.168.223.131:/data/k8s /var/lib/kubelet/pods/6f9acd8d-8d26-4245-8e6f-667ad8147fc9/volumes/kubernetes.io~nfs/nfs-client-root Output: mount: /var/lib/kubelet/pods/6f9acd8d-8d26-4245-8e6f-667ad8147fc9/volumes/kubernetes.io~nfs/nfs-client-root: bad option; for several filesystems (e.g. nfs, cifs) you might need a /sbin/mount.<type> helper program.
```

那极有可能是因为某个节点上没有安装**nfs-common**

导致无法正常挂载nfs，以至于pod部署无法在挂载的nfs卷中访问

通过如下命令安装

`apt install nfs-common -y`

![](https://web.yellowei.com/wordpress/wp-content/uploads/2022/03/img_k8s_nfs_error_handle-scaled.jpg)

### 修改kube-apiserver配置

因为之前遇到坑了，所以这一步之前放到正确的执行顺序上来讲，也及时提前告知坑点

如果我们没修改apiserver的配置，后续部署了StorageClass之后，再部署PVC的时候，会发现PVC一直处于Pending状态

当时没记录，不好复现了，请参考
`K8s 1.20x版本nfs动态存储报错 persistentvolume-controller waiting for a volume to be created, either by external provisioner "qgg-nfs-storage" or manually created by system administrator`[http://www.manongjc.com/detail/25-qebmmhrfcrfmqfq.html](http://www.manongjc.com/detail/25-qebmmhrfcrfmqfq.html "http://www.manongjc.com/detail/25-qebmmhrfcrfmqfq.html")

```shell
[root@k8s-matser01 nfs.rbac]# kubectl get pvc
NAME         STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS          AGE
test-claim   Pending                                      managed-nfs-storage   5s
[root@k8s-matser01 nfs.rbac]# kubectl describe pvc test-claim 
Name:          test-claim
Namespace:     default
StorageClass:  managed-nfs-storage
Status:        Pending
Volume:        
Labels:        <none>
Annotations:   volume.beta.kubernetes.io/storage-class: managed-nfs-storage
               volume.beta.kubernetes.io/storage-provisioner: qgg-nfs-storage
Finalizers:    [kubernetes.io/pvc-protection]
Capacity:      
Access Modes:  
VolumeMode:    Filesystem
Used By:       <none>
Events:
  Type    Reason                Age               From                         Message
  ----    ------                ----              ----                         -------
  Normal  ExternalProvisioning  8s (x2 over 13s)  persistentvolume-controller  waiting for a volume to be created, either by external provisioner "qgg-nfs-storage" or manually created by system administrator
```

解决方法就是修改apiserver配置
```shell
[root@k8s-matser01 nfs.rbac]# vim /etc/kubernetes/manifests/kube-apiserver.yaml
apiVersion: v1
···
    - --tls-private-key-file=/etc/kubernetes/pki/apiserver.key
    - --feature-gates=RemoveSelfLink=false # 添加这个配置
```

重启master节点或者apiserver服务后即可生效

接下来就可以快乐的创建SC和PVC了

### 部署默认的StorageClass

#### 1. 通过命令 `kubectl apply -f 对应的yaml文件名称  ` 部署SC

#### 2. 如果有兴趣的同学，可以下载GitHub上面的四个文件

运行以下脚本，自动下载
```shell
for file in class.yaml deployment.yaml rbac.yaml test-claim.yaml ; do wget https://raw.githubusercontent.com/kubernetes-incubator/external-storage/master/nfs-client/deploy/$file ; done
```

* class.yaml 其实就是StorageClass
```shell
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: managed-nfs-storage #这里的名称要和provisioner配置文件中的环境变量PROVISIONER_NAME保持一致
  annotations:
          "storageclass.kubernetes.io/is-default-class": "true"   #添加此句将sc设为默认
provisioner: yellowei-nfs-storage # or choose another name, must match deployment's env PROVISIONER_NAME'
parameters:
  archiveOnDelete: "false"
```

* deployment.yaml 
用于部署一个自动构建存储空间的deployment, 理解为provisioner
```shell
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-client-provisioner
  labels:
    app: nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: default #与RBAC文件中的namespace保持一致
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: nfs-client-provisioner
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: nfs-client-provisioner
      containers:
        - name: nfs-client-provisioner
          image: quay.io/external_storage/nfs-client-provisioner:latest
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: yellowei-nfs-storage #provisioner名称,请确保该名称与class.yaml文件中的provisioner名称保持一致
            - name: NFS_SERVER
              value: 192.168.223.131 #刚刚在worker节点NFS Server IP地址	
            - name: NFS_PATH
              value: /data/k8s  #刚刚在worker节点配置的NFS挂载卷
      volumes:
        - name: nfs-client-root
          nfs:
            server: 192.168.223.131 #刚刚在worker节点NFS Server IP地址
            path: /data/k8s #刚刚在worker节点配置的NFS挂载卷
```

如果拉取镜像失败，可以替换中科大镜像加速：quay.mirrors.ustc.edu.cn

或者我的私有私有仓库：`registry.cn-shenzhen.aliyuncs.com/yellowei/nfs-client-provisioner`

* rbac.yaml 
配置account及相关权限

```shell
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: default #根据实际环境设定namespace,下面类同
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: nfs-client-provisioner-runner
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "update", "patch"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: run-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    # replace with namespace where provisioner is deployed
    namespace: default
roleRef:
  kind: ClusterRole
  name: nfs-client-provisioner-runner
  apiGroup: rbac.authorization.k8s.io
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: default
rules:
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: default
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    # replace with namespace where provisioner is deployed
    namespace: default
roleRef:
  kind: Role
  name: leader-locking-nfs-client-provisioner
  apiGroup: rbac.authorization.k8s.io
```

* test-claim.yaml
一个用于测试SC是否能成功动态分配的pvc案例
```shell
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: test-claim
  annotations:
    volume.beta.kubernetes.io/storage-class: "managed-nfs-storage"
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
  storageClassName: # 不指定SC自动使用默认的， 即使在不同的namespace
```

不指定SC自动使用默认的， 即使在不同的namespace
不指定SC自动使用默认的， 即使在不同的namespace
不指定SC自动使用默认的， 即使在不同的namespace

当部署玩test-claim的pvc时，可得到如下输出
```shell
[root@k8s-master-node1 /home/yellowei/nfs-sc]# kubectl get pvc
NAME         STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS          AGE
test-claim   Bound    pvc-d191719f-1aef-4f66-9022-138ddcdcaa89   5Gi        RWX            managed-nfs-storage   67m
[root@k8s-master-node1 /home/yellowei/nfs-sc]# kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                                             STORAGECLASS          REASON   AGE
pvc-d191719f-1aef-4f66-9022-138ddcdcaa89   5Gi        RWX            Delete           Bound    default/test-claim                                                managed-nfs-storage            67m
```



结合pod来验证SC是否成功

```shell
kind: Pod
apiVersion: v1
metadata:
  name: test-pod
spec:
  containers:
  - name: test-pod
    image: busybox:1.24
    command:
      - "/bin/sh"
    args:
      - "-c"
      - "touch /mnt/SUCCESS && exit 0 || exit 1"   #创建一个SUCCESS文件后退出
    volumeMounts:
      - name: nfs-pvc
        mountPath: "/mnt"
  restartPolicy: "Never"
  volumes:
    - name: nfs-pvc
      persistentVolumeClaim:
        claimName: test-claim  #与PVC名称保持一致
```

部署上面的pod之后，会有如下输出

```shell
[root@k8s-master-node1 /home/yellowei/nfs-sc]# kubectl get pod
NAME                                      READY   STATUS      RESTARTS   AGE
ingress-demo-app-694bf5d965-6dm2g         1/1     Running     0          4h17m
ingress-demo-app-694bf5d965-tf89f         1/1     Running     0          4h17m
nfs-client-provisioner-6566b96985-mmtmp   1/1     Running     0          69m
test-pod                                  0/1     Completed   0          67m
```

其中test-pod就是测试pvc的pod， 如果status为completed， 则会在worker节点的nfs共享目录下（/data/k8s）的pv目录下，产生一个success文件

```shell
yellowei@k8s-worker-node1:/data/k8s$ ls
default-test-claim-pvc-d191719f-1aef-4f66-9022-138ddcdcaa89
yellowei@k8s-worker-node1:/data/k8s$ cd default-test-claim-pvc-d191719f-1aef-4f66-9022-138ddcdcaa89/
yellowei@k8s-worker-node1:/data/k8s$ ls
SUCCESS
yellowei@k8s-worker-node1:/data/k8s/default-test-claim-pvc-d191719f-1aef-4f66-9022-138ddcdcaa89$ ls -l
total 0
-rw-r--r-- 1 nobody nogroup 0 Mar  7 00:54 SUCCESS
```

至此，我们的SC动态方案已经部署完成
