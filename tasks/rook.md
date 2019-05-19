
# ROOK
 Rook是Kubernetes的开源云本地存储协调器，为各种存储解决方案提供平台，框架和支持，以便与云原生环境本地集成。Rook也是云原生计算基金会(CNCF)的孵化级项目[Rook官网:](https://rook.io),我们通过配置Rook的后端存储引擎选择使用[CEPH](https://ceph.com/)

### 安装Rook
```bash
cd $HOME
git clone https://github.com/rook/rook.git
------------------
给节点打标签:
运行ceph-mon的节点打上：ceph-mon=enabled
kubectl label nodes 192.168.10.243 ceph-mon=enabled
运行ceph-osd的节点，也就是存储节点，打上：ceph-osd=enabled
kubectl label nodes 192.168.10.243 ceph-osd=enabled
运行ceph-mgr的节点，打上：ceph-mgr=enabled
#mgr只能支持一个节点运行，这是ceph跑k8s里的局限
kubectl label nodes 192.168.10.243 ceph-mgr=enabled
---------------------

cd rook/cluster/examples/kubernetes/ceph
$ kubectl create -f common.yaml
$ kubectl create -f operator.yaml
$ kubectl -n rook-ceph get pod -o wide
NAME                                  READY   STATUS    RESTARTS   AGE
rook-ceph-agent-lkl8n                 1/1     Running   0          13s
rook-ceph-operator-7b976856bf-t979h   1/1     Running   0          15s
rook-discover-n9b7t                   1/1     Running   0          13s
```
配置 cluster：
```yaml
network:
  # toggle to use hostNetwork
  hostNetwork: true
-----
nodes:
- name: "192.168.10.243"
  devices:
  - name: "sdb"
```
```bash
$ kubectl create -f cluster.yaml
```

### 验证
```bash
$ kubectl -n rook-ceph get pod
NAME                                         READY   STATUS      RESTARTS   AGE
rook-ceph-agent-lkl8n                        1/1     Running     0          15m
rook-ceph-mgr-a-54f4f69d8d-j9lg2             1/1     Running     0          11m
rook-ceph-mon-a-759c494988-wsb84             1/1     Running     0          12m
rook-ceph-operator-7b976856bf-t979h          1/1     Running     0          15m
rook-ceph-osd-prepare-192.168.10.243-9d7rp   0/2     Completed   0          11m
rook-discover-n9b7t                          1/1     Running     0          15m
```

### 访问 dashborad
```bash
$ kubectl  create -f dashboard-external-https.yaml
```
![ceph-login](/images/ceph-login.png)
```bash
$ kubectl -n rook-ceph get secret rook-ceph-dashboard-password -o jsonpath="{['data']['password']}" | base64 --decode && echo
LRDPOtup5S
```
密码拿到之后， 使用用户名 admin 和密码进行登录即可

### 命令行操作
创建客户端工具容器:
```bash
$ kubectl  create  -f toolbox.yaml
$ kubectl -n rook-ceph get pod -l "app=rook-ceph-tools"
```
进入工具容器：
```bash
$ kubectl -n rook-ceph exec -it $(kubectl -n rook-ceph get pod -l "app=rook-ceph-tools" -o jsonpath='{.items[0].metadata.name}') bash
[root@node02 /]# ceph status
  cluster:
    id:     021c227d-a608-401f-a992-fb10cbe5e2ad
    health: HEALTH_OK

  services:
    mon: 1 daemons, quorum a (age 32m)
    mgr: a(active, since 31m)
    osd: 0 osds: 0 up, 0 in

  data:
    pools:   0 pools, 0 pgs
    objects: 0 objects, 0 B
    usage:   0 B used, 0 B / 0 B avail
    pgs:     

[root@node02 /]# ceph osd status
+----+------+------+-------+--------+---------+--------+---------+-------+
| id | host | used | avail | wr ops | wr data | rd ops | rd data | state |
+----+------+------+-------+--------+---------+--------+---------+-------+
+----+------+------+-------+--------+---------+--------+---------+-------+
[root@node02 /]# ceph df
RAW STORAGE:
    CLASS     SIZE     AVAIL     USED     RAW USED     %RAW USED
    TOTAL      0 B       0 B      0 B          0 B             0

POOLS:
    POOL     ID     STORED     OBJECTS     USED     %USED     MAX AVAIL
[root@node02 /]# rados df
POOL_NAME USED OBJECTS CLONES COPIES MISSING_ON_PRIMARY UNFOUND DEGRADED RD_OPS RD WR_OPS WR USED COMPR UNDER COMPR

total_objects    0
total_used       0 B
total_avail      0 B
total_space      0 B
# ceph df
GLOBAL:
    SIZE        AVAIL       RAW USED     %RAW USED
    20.0GiB     19.0GiB      1.00GiB          5.00
POOLS:
    NAME     ID     USED     %USED     MAX AVAIL     OBJECTS
    
```

#### 创建 pool:
```bash
# ceph osd pool create bmw 64
# ceph osd pool set bmw size 5
#  ceph osd lspools            
1 bmw
```
#### 删除pool
```bash
# ceph osd pool delete bmw bmw --yes-i-really-really-mean-it
pool 'bmw' removed
提示：删除 pool 的时候要连续写两次pool的name
```

#### 设置POOL配额:
```bash
# ceph osd pool set-quota pool max_objects 100 #最大100个对象
# ceph osd pool set-quota pool1 max_bytes $((10 * 1024 * 1024 * 1024))    #容量大小最大为10G
```
#### 查看POOL状态信息
```bash
rados df
POOL_NAME USED OBJECTS CLONES COPIES MISSING_ON_PRIMARY UNFOUND DEGRADED RD_OPS  RD WR_OPS  WR USED COMPR UNDER COMPR
pool       0 B       0      0      0                  0       0        0      0 0 B      0 0 B        0 B         0 B
total_objects    0
total_used       0 B
total_avail      0 B
total_space      0 B
```

#### 创建用户
```bash
ceph auth get-or-create client.bmw mon 'allow r' osd 'allow class-read object_prefix rbd_children, allow rwx pool=bmw' -o ceph.client.bmw.keyring

[root@node02 /]# ceph auth get-key client.admin
AQC3nN9cYxRkLxAAUwyPVoB1iPdVSSkssxFgUg==
[root@node02 /]# ceph auth get-key client.bmw  
AQBFq99cokBPHBAA5DVD6kC/DYlBxst2+r9S0w==
```
正常来说，有了 pool的名字和 key， 我们就可以通过 k8s 访问ceph了；

删除工具集容器：
```bash
$ kubectl -n rook-ceph delete deployment rook-ceph-tools
```

####  清理集群
```bash
kubectl delete -f common.yaml
kubectl delete -f operator.yaml
kubectl delete -f cluster.yaml
-----
kubectl delete -f service-monitor.yaml
kubectl delete -f prometheus.yaml
kubectl delete -f prometheus-service.yaml
kubectl delete -f https://raw.githubusercontent.com/coreos/prometheus-operator/v0.26.0/bundle.yaml
```
