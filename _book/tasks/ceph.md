# ceph 容器化

- 拉取新版 ceph 镜像
```bash
$ docker pull ceph/daemon:latest
```

- 调整内核参数
```bash
cat >> /etc/sysctl.conf << EOF
kernel.pid_max=4194303
vm.swappiness = 0
EOF
sysctl -p
```

read_ahead, 通过数据预读并且记载到随机访问内存方式提高磁盘读操作，根据一些Ceph的公开分享，8192是比较理想的值
```bash
echo "8192" > /sys/block/sda/queue/read_ahead_kb
```

I/O Scheduler，关于I/O Scheculder 的调整，简单说SSD要用noop，SATA/SAS使用deadline
```bash
echo "deadline" > /sys/block/sd[x]/queue/scheduler
echo "noop" > /sys/block/sd[x]/queue/scheduler
```

#### 配置命令：
由于我们是容器化部署的，ceph的客户端也在容器内部，所以我们需要把容器内的 ceph 命令 alias 到本地，方便使用，其他 ceph 相关命令也可以参考添加：
```bash
echo 'alias ceph="docker exec mon ceph"' >> /etc/profile
source /etc/profile
```

#### 部署 ceph
启动MON:
```bash
docker run -d --net=host \
    --name=ceph-mon \
    --restart=always \
    -v /etc/localtime:/etc/localtime \
    -v /home/ceph/etc:/etc/ceph \
    -v /home/ceph/lib:/var/lib/ceph \
    -v /home/ceph/logs:/var/log/ceph \
    -e MON_IP=10.86.77.36 \
    -e CEPH_PUBLIC_NETWORK=10.86.0.0/16 \
    ceph/daemon:latest mon
```
***PS:***
```
MON_IP 是集群所有Mon节点的ip，要写全， 中间用逗号分隔；如果IP是跨网段的，CEPH_PUBLIC_NETWORK 的配置必须覆盖所有网段；
```
启动OSD:
```bash
docker run -d \
    --name=osd_data1 \
    --net=host \
    --restart=always \
    --privileged=true \
    --pid=host \
    -v /etc/localtime:/etc/localtime \
    -v /home/ceph/etc:/etc/ceph \
    -v /home/ceph/lib:/var/lib/ceph \
    -v /home/ceph/logs:/var/log/ceph \
    -v /home/data/osd1:/var/lib/ceph/osd \
    ceph/daemon:latest osd_directory
```

启动MDS:
```bash
docker run -d \
    --net=host \
    --name=mds \
    --restart=always \
    --privileged=true \
    -v /etc/localtime:/etc/localtime \
    -v /home/ceph/etc:/etc/ceph \
    -v /home/ceph/lib/:/var/lib/ceph/ \
    -v /data/ceph/logs/:/var/log/ceph/ \
    -e CEPHFS_CREATE=0 \  
    -e CEPHFS_METADATA_POOL_PG=512 \
    -e CEPHFS_DATA_POOL_PG=512 \
    ceph/daemon:latest mds
```
注解：
```
CEPHFS_CREATE: 0表示不自动创建文件系统（推荐），1表示自动创建
```
docker run -d --net=host --name=osd1 -v /etc/ceph:/etc/ceph -v /home/ceph/lib:/var/lib/ceph -v /dev:/dev -v /home/data/osd1:/var/lib/ceph/osd --privileged=true ceph/daemon:latest osd_directory
