
# ceph-deploy

#### yum 安装 ceph-deploy 
```bash
yum install ceph-deploy
```

#### 配置yum源
```
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
```
#### 部署ceph组件:
```bash
ceph-deploy install --no-adjust-repos kube-master01
....
[kube-master01][DEBUG ] Complete!
[kube-master01][INFO  ] Running command: ceph --version
[kube-master01][DEBUG ] ceph version 12.2.12 (1436006594665279fe734b4c15d7e08c13ebd777) luminous (stable)

# check ceph version 
ceph -v
ceph version 12.2.12 (1436006594665279fe734b4c15d7e08c13ebd777) luminous (stable)
```
------------------------

添加ceph集群:
```
ceph-deploy new --public-network 192.168.10.0/24  kube-master01
```

添加mons:
```
ceph-deploy mon create kube-master01
初始化monitor以及生成key
ceph-deploy mon create-initial
复制配置文件和密码到管理节点的/etc/ceph目录下,允许一主机以管理员权限执行 Ceph 命令
ceph-deploy admin kube-master01
```
创建manager守护进程
```bash
ceph-deploy mgr create kube-master01
```

添加osd
```bash
# ceph-deploy osd create --data /dev/sdb kube-master01
# ceph osd crush tunables jewel
# ceph osd crush reweight-all  
```
查看集群健康状态:
```bash
ceph health
ceph -s
```
为了能使用CephFS,这里将 kube-master01 设置为 metadata 服务器:
```bash
ceph-deploy mds create kube-master01
ceph mds stat
```

验证集群状态：
```bash
# ceph -s
  cluster:
    id:     087b8803-261d-43a7-9666-69771e2eca5f
    health: HEALTH_OK

  services:
    mon: 1 daemons, quorum kube-master01
    mgr: kube-master01(active)
    osd: 1 osds: 1 up, 1 in

  data:
    pools:   0 pools, 0 pgs
    objects: 0 objects, 0B
    usage:   1.00GiB used, 19.0GiB / 20.0GiB avail
    pgs:     
# ceph osd status
+----+---------------+-------+-------+--------+---------+--------+---------+-----------+
| id |      host     |  used | avail | wr ops | wr data | rd ops | rd data |   state   |
+----+---------------+-------+-------+--------+---------+--------+---------+-----------+
| 0  | kube-master01 | 1024M | 18.9G |    0   |     0   |    0   |     0   | exists,up |
+----+---------------+-------+-------+--------+---------+--------+---------+-----------+
```

#### 创建 pool:
```bash
# ceph osd pool create bmw 64
# ceph osd pool set bmw size 10
#  ceph osd lspools     
1 bmw
```
创建用户:
```bash
# ceph auth get-or-create client.bmw mon 'allow r' osd 'allow rwx pool=bmw' -o ceph.client.bmw.keyring
# ceph auth get-key client.admin
AQBG+N9cZT/TGhAA2CRtHHJUMnmhSBIYFWQWcg==
# ceph auth get-key client.bmw  
AQDh/N9cAqRsAxAAau/WQ3zlVkPoSjn42THfJA==
```
正常来说，有了 pool的名字和 key， 我们就可以通过 k8s 访问ceph了；

#### 创建 image
```bash
# rbd create --size 1 bmw/testimage --id admin -m 192.168.10.231:6789 --key=AQBG+N9cZT/TGhAA2CRtHHJUMnmhSBIYFWQWcg==
# rbd ls bmw
```

-----------------

清理集群配置信息:
```
 rm -rf /etc/ceph/*
 rm -rf /var/lib/ceph/*/*
 rm -rf /var/log/ceph/*
 rm -rf /var/run/ceph/*
```

#### TS
如果创建image时报错是关于 featrure missing 400000000000000的，可以通过如下方式解决：
```bash
# ceph osd crush tunables jewel
# ceph osd crush reweight-all  
```

***PS:*** 客户端和服务端版本要一致、要一致、要一致；
