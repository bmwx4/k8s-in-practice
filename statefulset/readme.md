# StatefulSet Overview

**Deployment用于部署无状态服务，StatefulSet 用来部署有状态服务**

那具体什么场景需要使用StatefulSet 呢？官方的建议是，如果你部署的应用满足以下一个或多个部署需求，则建议使用StatefulSet。
```
稳定的、唯一的网络标识。
稳定的、持久的存储。
有序的、优雅的部署和伸缩。
有序的、优雅的删除和停止。
有序的、自动的滚动更新。
```
***稳定*** 主要体现在 Pod 发生 re-schedule或rebuild 后仍然要保持之前的网络标识和持久化存储。这里所说的网络标识包括 hostname、集群内DNS中该Pod对应的A Record，但 **并不能保证Pod re-schedule之后IP不变**。
如果想保持Pod IP不变，可以借助Pod hostname，然后定制IPAM获取固定的Pod IP。因此借助 StatefulSet 稳定的唯一的网络标识特性，是能够比较轻松的实现 Pod 的固定IP需求的，如果使用Deployment，情况将将会变得复杂；
