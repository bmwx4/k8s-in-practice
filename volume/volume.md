# Volume
在某些业务场景中，业务希望启动时加载初始化数据，而且业务也希望，当容器重启之后，能够从之前的容器结束位置继续运行，因为先前容器生产的数据，不能被丢失,它将
作为本次容器启动时的状态数据。大家应该都知道，在默认情况下Docker容器中的数据都是非持久化的，容器消亡后数据也会消失，Docker提供了Volume机制以便实现数据的持久化。
Kubernetes中Volume的概念与Docker中的Volume类似，同样需要一个独立于容器镜像文件系统的存储区域，该存储区域可以是临时的、可以是本地宿主机上的磁盘，或者是网络共享磁盘等；
最终可以通过volume mount的方式，挂载到容器内部任意路径；
Kubernetes提供了非常丰富的Volume类型，下面是一些常用的Volume类型：

- emptyDir and gitRepo
- hostPath
- nfs
- ceph-rbd
- cephfs
- secret
- downwardAPI
- persistentVolumeClaim
