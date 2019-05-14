# ceph deployment

#### Ceph Mon 节点安装
Ceph的MON是一个集群映射图的“主副本”，客户端只需通过连接到MON就可以了解Ceph-MON、Ceph的OSD守护进程，和Ceph的元数据服务器的位置。

ceph配置如下：
```
# cat /etc/ceph/ceph.conf
[global]

fsid = 695b3afb-f560-434f-a90a-f611e1f23638

mon initial members = node02

mon host = 192.168.10.243

auth cluster required = cephx

auth service required = cephx

auth client required = cephx



osd pool default size = 2
public network = 192.168.10.0/24

cluster network = 192.168.10.0/24

osd journal size = 100
osd pool default pg num = 100
osd pool default pgp num = 100

osd_memory_target = 2838712320
osd_memory_base = 1445570560
osd_memory_cache_min = 2142141440
```

启动Mon：
```bash
docker run \
-d --net=host  \
--name=mon \
-v /etc/ceph:/etc/ceph \
-v /var/lib/ceph/:/var/lib/ceph \
-e MON_IP=192.168.10.243 \
-e CEPH_PUBLIC_NETWORK=192.168.10.0/24 \
ceph/daemon:luminous \
Mon
```

验证Mon：
```bash
docker ps|grep ceph|grep mon
ps aux|grep ceph|grep mon
docker logs -f mon
docker exec mon ceph -s
```

#### Ceph OSD节点安装
Ceph OSD:Ceph Object Storage Device 主要功能包括:存储数据,副本数据处理,数据恢复,数据回补,平衡数据分布,并将数据相关的一些儿监控信息提供给至少2个Ceph OSD,才能有效保存两份数据.OSD节点安装有两种方案，一种是在节点上挂载全新的硬盘设备，第二种是将已安装好的系统的指定目录作为OSD。
我们这里选用第二种方式，只是因为方便。
```bash
docker run \
-d --net=host \
--name=osd \
--privileged=true \
--pid=host \
-v /etc/ceph:/etc/ceph \
-v /var/lib/ceph/:/var/lib/ceph/ \
-v /dev/:/dev/ \
-e OSD_FORCE_ZAP=1 \
-e OSD_TYPE=directory   \
ceph/daemon:luminous \
Osd
```
