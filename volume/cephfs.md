# cephfs
cephfs 是共享文件系统；
cephfs的使用方式:
```
cephfs kernel module
cephfs-fuse
```
![cephfs](/images/cephfs.png)
从上面的架构图中可以看到，cephfs-fuse的方式比cephfs kernel module方式在性能方面会差一些。毕竟ceph-fuse是用户态的库，反过来说，从全局稳定性来说，如果ceph 集群出现故障，ceph-fuse的影响是单个实例，而 kernel方式影响的是整个宿主，即(node节点)；所以，推荐是用ceph-fuse方式；

#### 创建cephfs
在ceph master上创建cephfs 集群:
```
[root@kube-master01 cluster]# ceph osd pool create cephfs_data 3
pool 'cephfs_data' created
[root@kube-master01 cluster]# ceph osd pool create cephfs_metadata 3
pool 'cephfs_metadata' created
[root@kube-master01 cluster]# ceph fs new cephfs cephfs_metadata cephfs_data
new fs with metadata pool 3 and data pool 2
[root@kube-master01 cluster]# ceph fs ls
name: cephfs, metadata pool: cephfs_metadata, data pools: [cephfs_data ]
```
#### kernel方式挂载测试
```bash
# mount -t ceph 192.168.10.231:6789:/ /mnt/cephfs -o name=admin,secret=AQBG+N9cZT/TGhAA2CRtHHJUMnmhSBIYFWQWcg==
# mount|grep cephfs
192.168.10.231:6789:/ on /mnt/cephfs type ceph (rw,relatime,name=admin,secret=<hidden>,acl,wsize=16777216)
```
#### ceph-fuse 方式挂载测试
```bash
# ceph-fuse -k /etc/ceph/ceph.client.admin.keyring -m 192.168.10.231 /cephfs
# mount |grep ceph
ceph-fuse on /cephfs type fuse.ceph-fuse (rw,nosuid,nodev,relatime,user_id=0,group_id=0,allow_other)
```

#### 创建pod
在pod应用使用 ceph-fuse 方式挂载 cephfs:
```yaml
# cephfs-test.yaml
apiVersion: v1
kind: Pod
metadata:
  name: cephfs-test
spec:
  containers:
  - name: cephfs-c
    image: nginx:latest
    imagePullPolicy: IfNotPresent
    volumeMounts:
    - mountPath: "/mnt/cephfs"
      name: cephfs
  volumes:
  - name: cephfs
    cephfs:
      monitors:
      - 192.168.10.231:6789
      user: "admin"
      secretRef:
        name: ceph-admin-secret
      readOnly: false
```
查看结果:
```bash
# kubectl  get pod -o wide
NAME                                READY   STATUS    RESTARTS   AGE    IP            NODE             NOMINATED NODE
cephfs-test                         1/1     Running   0          11s    172.30.7.8    192.168.10.243   <none>

# kubectl  exec -it cephfs-test bash

root@cephfs-test:/# mount |grep ceph
ceph-fuse on /mnt/cephfs type fuse.ceph-fuse (rw,nosuid,nodev,relatime,user_id=0,group_id=0,allow_other)
```
#### 总结
cephfs 支持多个客户端同时写入，而rbd同时只能有一个写者；而且 kubernetes 官方并没有对cephfs提供类似ceph RBD动态分配PV的storageClass的功能；但是社区提供了一个 **[cephfs-provisioner](https://github.com/kubernetes-incubator/external-storage/tree/master/ceph/cephfs)** 来实现这个功能,目前用的还不多，慎用；
