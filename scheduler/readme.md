# kube-scheduler
当要在集群中提交运行的工作负载时，调度程序会确定要在哪个node上放置与工作负载关联的 Pod。调度程序可以在任何满足 Pod 的 CPU、内存和自定义资源要求的节点上自由放置 Pod。但是， 通过 pod spec配置，也可以进行一些特殊node的选择，比如我们早期用过的 nodeSelector ；但是 nodeSelector 还是不够灵活，满足不了个性化调度需求，所以，下面将介绍如下几种调度机制:

```
nodeAffinity: 节点亲和性
podAffinity:  pod 亲和性
podAntiAffinity: pod 反亲和性
taints-tolerence: 污点与容忍
```
