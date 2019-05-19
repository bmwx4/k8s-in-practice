# ceph rbd
ceph-rbd 是持久化对象存储

#### 安装 ceph 客户端
```bash
添加 ceph-fuse  yum 源
vim  /etc/yum.repos.d/ceph.repo
目标版本:
# ceph --version
ceph version 12.2.12 (1436006594665279fe734b4c15d7e08c13ebd777) luminous (stable)
yum源:
[root@node02 ~]# cat /etc/yum.repos.d/ceph.repo
[ceph]
name=Ceph packages for $basearch
baseurl=http://mirrors.163.com/ceph/rpm-luminous/el7/$basearch
enabled=1
priority=2
gpgcheck=1
gpgkey=https://download.ceph.com/keys/release.asc

[ceph-noarch]
name=Ceph noarch packages
baseurl=http://mirrors.163.com/ceph/rpm-luminous/el7/noarch
enabled=1
priority=2
gpgcheck=1
gpgkey=https://download.ceph.com/keys/release.asc

[ceph-source]
name=Ceph source packages
baseurl=http://mirrors.163.com/ceph/rpm-luminous/el7/SRPMS
enabled=0
priority=2
gpgcheck=1
gpgkey=https://download.ceph.com/keys/release.asc

[root@node02 ~]# yum clean all && yum makecache
[root@node02 ~]# yum install ceph ceph-fuse -y
```


#### 创建访问ceph集群的 secret
```bash
admn-secret:
$ ceph auth get-key client.admin
$ export CEPH_ADMIN_SECRET='AQBG+N9cZT/TGhAA2CRtHHJUMnmhSBIYFWQWcg=='
$ kubectl create secret generic ceph-admin-secret --type="kubernetes.io/rbd" --from-literal=key=$CEPH_ADMIN_SECRET
bmw-secret:
$ ceph auth get-key client.bmw
$ export CEPH_USER_SECRET='AQDh/N9cAqRsAxAAau/WQ3zlVkPoSjn42THfJA=='
$ kubectl create secret generic ceph-bmw-secret --type="kubernetes.io/rbd" --from-literal=key=$CEPH_USER_SECRET
```

### 静态PV
![static](/images/static-pv.jpg)
**创建一个image:**  
```bash
$ rbd create bmw/ceph-image -s 20
```
**创建一个PV:**
```yaml
#ceph-pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: ceph-pv
spec:
  capacity:
    storage: 20Mi
  accessModes:
    - ReadWriteOnce
  rbd:
    monitors:
      - 192.168.10.231:6789
    pool: bmw
    image: ceph-image
    user: admin
    secretRef:
      name: ceph-admin-secret
    fsType: ext4
    readOnly: false
  persistentVolumeReclaimPolicy: Recycle
```
accessModes 支持三种模式
```
ReadWriteOnce(RWO)：是最基本的方式，可读可写，但只支持被单个Pod挂载。
ReadOnlyMany(ROX)：可以以只读的方式被多个Pod挂载。
ReadWriteMany(RWX)：这种存储可以以读写的方式被多个Pod共享。
```
PV 是以 plugin 的形式来提供支持的， 考虑到私有云的使用场景， 排除掉azure， aws，gce等公有云厂商绑定的plugin， 有9个插件值得关注。这些 plugin 所支持的accessmode 是不同的。 分别是：
![pv-plugin.jpg](/images/pv-plugin.jpg)
所以，在选择 accessModes 时不要盲目选择；

Reclaim Policy支持以下三种:
```
Retain: 如果用户删除 PVC， PV 不会被删除,需要手动清理。相反，它将变为 Released 状态，表示所有的数据可以被手动恢复。
Recycle: 删除数据，等同于rm -rf /thevolume/* (只有nfs,hostpath支持)
Delete: 对于动态配置的 PV 来说，默认回收策略为"Delete"。这表示当用户删除对应的 PVC 时，动态配置的 PV 将被自动删除,相应的 image 也被删除了；所以，如果 volume 包含重要数据时，这种自动行为可能是不合适的。
```
执行并查看效果:
```bash
$ kubectl create -f ceph-pv.yaml
$ kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM                    STORAGECLASS       REASON   AGE
ceph-pv                                    20Mi       RWO            Recycle          Available                                                        14s
```
四个主要状态(PV STATUS)
```
Available: 资源可用， 还没有被PVC绑定
Bound: 被 PVC 绑定
Released: 绑定的PVC被删除了，但是还没有被集群重声明
Failed: 自动回收失败
```

**创建一个PVC:**  
```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: ceph-claim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Mi
```
PVC需要指定访问模式和存储的大小，当前只支持这两个属性;一个PVC绑定到一个PV上。一旦PV被绑定到PVC上就不能被其它的PVC所绑定,它们是一对一的关系。但是多个Pod可以使用同一个PVC进行卷的挂载(但不推荐这么做)。  
执行创建并验证:
```bash
# kubectl  create -f pvc.yaml
# kubectl  get pv,pvc
NAME                                                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                    STORAGECLASS       REASON   AGE
persistentvolume/ceph-pv                                    20Mi       RWO            Recycle          Bound    default/ceph-claim                                   5m23s
persistentvolume/pvc-4a3c172d-7971-11e9-8bb5-000c29a5444f   2Mi        RWO            Delete           Bound    default/ceph-rdb-claim   dynamic-ceph-rdb            130m

NAME                                   STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS       AGE
persistentvolumeclaim/ceph-claim       Bound    ceph-pv                                    20Mi       RWO                               16s
persistentvolumeclaim/ceph-rdb-claim   Bound    pvc-4a3c172d-7971-11e9-8bb5-000c29a5444f   2Mi        RWO            dynamic-ceph-rdb   130m
```
**创建一个pod应用PVC**
```yaml
#nginx-static-pv-test.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-static-pv-test
spec:
  containers:
  - name: nginx
    image: nginx:latest
    imagePullPolicy: IfNotPresent
    volumeMounts:
      - name: nginx-static-pv-test
        mountPath: /data/
        readOnly: false
  volumes:
  - name: nginx-static-pv-test
    persistentVolumeClaim:
      claimName: ceph-claim
```
执行创建并验证:
```bash
kubectl  create -f nginx-static-pv-test.yaml
# kubectl  get pod -o wide nginx-static-pv-test
NAME                   READY   STATUS    RESTARTS   AGE   IP           NODE             NOMINATED NODE
nginx-static-pv-test   1/1     Running   0          20s   172.30.7.9   192.168.10.243   <none>
# kubectl exec nginx-static-pv-test mount|grep rbd
/dev/rbd1 on /data type ext4 (rw,relatime,stripe=4096,data=ordered)
```
如上就是所谓的静态PV，无论是应用还是管理这些pv与pvc，都很麻烦。

--------------------

### 动态PV
![dynamic](/images/dynamic-pv.jpg)
#### 创建 storageClass
```bash
cat >storageclass-ceph-rdb.yaml<<EOF
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: dynamic-ceph-rdb
provisioner: kubernetes.io/rbd
parameters:
  monitors: 192.168.10.231:6789
  adminId: admin
  adminSecretName: ceph-admin-secret
  # adminSecretNamespace: kube-system
  pool: bmw
  userId: bmw
  userSecretName: ceph-bmw-secret
  fsType: ext4
  imageFormat: "2"
  imageFeatures: "layering"
EOF
$ kubectl  create  -f storageclass-ceph-rdb.yaml
```

#### 创建pvc测试
```bash
cat >ceph-rdb-pvc-test.yaml<<EOF
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: ceph-rdb-claim
spec:
  accessModes:     
    - ReadWriteOnce
  storageClassName: dynamic-ceph-rdb
  resources:
    requests:
      storage: 2Mi
EOF
$ kubectl apply -f ceph-rdb-pvc-test.yaml
```
查看效果:
```bash
[root@master01 ~]# kubectl  get pvc
NAME             STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS       AGE
ceph-rdb-claim   Bound    pvc-4a3c172d-7971-11e9-8bb5-000c29a5444f   2Mi        RWO            dynamic-ceph-rdb   3s
[root@master01 ~]# kubectl  get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                    STORAGECLASS       REASON   AGE
pvc-4a3c172d-7971-11e9-8bb5-000c29a5444f   2Mi        RWO            Delete           Bound    default/ceph-rdb-claim   dynamic-ceph-rdb            7s
```
说明已经创建成功;
创建个pod 测试一下:
```yaml
#nginx-rbd.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-rbd-test
spec:
  containers:
  - name: nginx
    image: nginx:latest
    imagePullPolicy: IfNotPresent
    volumeMounts:
      - name: nginx-rbd
        mountPath: /data/
        readOnly: false
  volumes:
  - name: nginx-rbd
    persistentVolumeClaim:
      claimName: ceph-rdb-claim
```
查看结果:
```bash
# kubectl  describe pod nginx-rbd-test
Events:
  Type    Reason                  Age   From                     Message
  ----    ------                  ----  ----                     -------
  Normal  Scheduled               26s   default-scheduler        Successfully assigned default/nginx-rbd-test to 192.168.10.243
  Normal  SuccessfulAttachVolume  25s   attachdetach-controller  AttachVolume.Attach succeeded for volume "pvc-4a3c172d-7971-11e9-8bb5-000c29a5444f"
  Normal  Pulled                  9s    kubelet, 192.168.10.243  Container image "nginx:latest" already present on machine
  Normal  Created                 9s    kubelet, 192.168.10.243  Created container
  Normal  Started                 8s    kubelet, 192.168.10.243  Started container
# kubectl  get pod nginx-rbd-test -owide
NAME             READY   STATUS    RESTARTS   AGE   IP           NODE             NOMINATED NODE
nginx-rbd-test   1/1     Running   0          84s   172.30.7.7   192.168.10.243   <none>
# kubectl  exec nginx-rbd-test  mount |grep rbd
/dev/rbd0 on /data type ext4 (rw,relatime,stripe=4096,data=ordered)
```

#### 写文件，重建容器进行测试
