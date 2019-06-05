# containerd
Containerd 是一个工业级标准的容器运行时，它强调简单性、健壮性和可移植性。Containerd 可以在宿主机中管理完整的容器生命周期：容器镜像的传输和存储、容器的执行和管理、存储和网络等。详细点说，Containerd 负责干下面这些事情：
```bash
管理容器的生命周期(从创建容器到销毁容器)
拉取/推送容器镜像
存储管理(管理镜像及容器数据的存储)
调用 runC 运行容器(与 runC 等容器运行时交互)
管理容器网络接口及网络
```

#### 架构演进
Kubernetes 的 Containerd 集成架构有两次重大改进，每一次都让整个体系更加稳定和高效。
![architecture](/images/architecture.png)

#### Install Containerd
```bash
可以源码编译:
wget https://github.com/containerd/containerd/archive/v1.2.4.zip
unzip v1.2.4.zip
也可以直接解压tar包到各个相关目录:
wget https://storage.googleapis.com/cri-containerd-release/cri-containerd-1.2.4.linux-amd64.tar.gz
tar --no-overwrite-dir -C / -xzf cri-containerd-${VERSION}.linux-amd64.tar.gz

解压之后的目录结构:
$ tree
├── etc
│   ├── crictl.yaml
│   └── systemd
│       └── system
│           └── containerd.service
├── opt
│   └── containerd
│       └── cluster...
└── usr
    └── local
        ├── bin
        │   ├── containerd
        │   ├── containerd-shim
        │   ├── containerd-shim-runc-v1
        │   ├── containerd-stress
        │   ├── crictl
        │   ├── critest
        │   └── ctr
        └── sbin
            └── runc
```

#### configFile
可以通过 containerd 命令初始化默认配置
```bash
# containerd config default > /etc/containerd/config.toml
```
内容如下:
```toml
root = "/var/lib/containerd"
state = "/run/containerd"
oom_score = 0

[grpc]
  address = "/run/containerd/containerd.sock"
  uid = 0
  gid = 0
  max_recv_message_size = 16777216
  max_send_message_size = 16777216

[debug]
  address = ""
  uid = 0
  gid = 0
  level = ""

[metrics]
  address = ""
  grpc_histogram = false

[cgroup]
  path = ""

[plugins]
  [plugins.cgroups]
    no_prometheus = false
  [plugins.cri]
    stream_server_address = "127.0.0.1"
    stream_server_port = "0"
    enable_selinux = false
    sandbox_image = "k8s.gcr.io/pause:3.1"
    stats_collect_period = 10
    systemd_cgroup = false
    enable_tls_streaming = false
    max_container_log_line_size = 16384
    [plugins.cri.containerd]
      snapshotter = "overlayfs"
      no_pivot = false
      [plugins.cri.containerd.default_runtime]
        runtime_type = "io.containerd.runtime.v1.linux"
        runtime_engine = ""
        runtime_root = ""
      [plugins.cri.containerd.untrusted_workload_runtime]
        runtime_type = ""
        runtime_engine = ""
        runtime_root = ""
    [plugins.cri.cni]
      bin_dir = "/opt/cni/bin"
      conf_dir = "/etc/cni/net.d"
      conf_template = ""
    [plugins.cri.registry]
      [plugins.cri.registry.mirrors]
        [plugins.cri.registry.mirrors."docker.io"]
          endpoint = ["https://registry-1.docker.io"]
    [plugins.cri.x509_key_pair_streaming]
      tls_cert_file = ""
      tls_key_file = ""
  [plugins.diff-service]
    default = ["walking"]
  [plugins.linux]
    shim = "containerd-shim"
    runtime = "runc"
    runtime_root = ""
    no_shim = false
    shim_debug = false
  [plugins.opt]
    path = "/opt/containerd"
  [plugins.restart]
    interval = "10s"
  [plugins.scheduler]
    pause_threshold = 0.02
    deletion_threshold = 0
    mutation_threshold = 100
    schedule_delay = "0s"
    startup_delay = "100ms"
```
[各字段解释](https://github.com/containerd/cri/blob/master/docs/config.md)
#### 配置kubelet 应用 Containerd
创建 /etc/systemd/system/kubelet.service.d/0-containerd.conf systemd drop-in 文件，内容如下:
```
[Service]                                                 
Environment="KUBELET_EXTRA_ARGS=--container-runtime=remote --runtime-request-timeout=15m --container-runtime-endpoint=unix:///run/containerd/containerd.sock"
```
执行如下命令生效:
```bash
# systemctl daemon-reload
# systemctl restart containerd
```
这样kubelet 就应用 containerd 了;

#### 运行效果
在宿主上查看pods:
![pods](/images/pods-result.png)
在宿主上查看容器:
![containers](/images/containers-result.png)
在容器内执行命令:
```bash
$ crictl exec d441df0c48e78 ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
2: eth0@if59: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN
    link/ether 8e:05:79:c8:13:b5 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet xxx.xxx.xxx.xxx/24 scope global eth0
       valid_lft forever preferred_lft forever
```

#### Docker 兼容
用户可以配置 Kubernetes的kubelet 来使用 Containerd 作为runtime， 但是， Containerd 同时还会给同一节点上的 Docker Engine 提供支撑的，比如下图。
![docker-containerd](/images/docker-containerd.png)

#### 参考
[getting-started](https://containerd.io/docs/getting-started/)

#### PS
如果 snapshotter 使用的是 overlayfs 的话，请注意下面的报错信息:
```
could not use snapshotter overlayfs in metadata plugin" error="/home/docker_rt/io.containerd.snapshotter.v1.overlayfs does not support d_type. If the backing filesystem is xfs, please reformat with ftype=1
.......
failed to load plugin io.containerd.grpc.v1.cri" error="failed to create CRI service: failed to find snapshotter "overlayfs""
```
[参考](https://linuxer.pro/2017/03/what-is-d_type-and-why-docker-overlayfs-need-it/)

正常情况下是这样的 ftype=1的情况:
```bash
$ xfs_info /home/
meta-data=/dev/md1               isize=256    agcount=32, agsize=21974144 blks
         =                       sectsz=4096  attr=2, projid32bit=1
         =                       crc=0        finobt=0 spinodes=0
data     =                       bsize=4096   blocks=703170816, imaxpct=5
         =                       sunit=128    swidth=384 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal               bsize=4096   blocks=343345, version=2
         =                       sectsz=4096  sunit=1 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
```

临时解决方案，可以把 snapshotter 的值改成 native；
