# kube-proxy
### kube-proxy、 Service以及 clusterIP
1. 集群中 pod 更新时，容器的IP地址可能发生变化，如果直接通过IP访问容器的话，会有很大的风险。
2. 进行集群扩容时，rc/rs 中会有新的 pod 创建出来，出现新的 ip 地址，我们需要一种更灵活的方式来访问 pod 的服务。
3. 进行集群缩容时，rc/rs 中会减少pod ，原ip已经被 IPam 回收不可访问。

针对上面的问题，kubernetes 的解决方案是 用 service 来抽象这个集群的后端有效pod；

***clusterIP***

每个服务都有一个固定的虚拟 ip（也被称为 cluster IP),这个IP
自动并且动态地绑定后面的 pod，所有的网络请求直接访问服务 ip，服务会自动向后端做转发。Service 除了提供稳定的对外访问方式之外，还能起到负载均衡（Load Balance）的功能，自动把请求流量分布到后端所有的服务上，服务可以做到对客户透明地进行水平扩展（scale），而实现service 这一特性的组件就是kube-proxy, kube-proxy运行在每个worker节点上，监听API Server中的service对象的变化，然后通过 iptables 来管理和实现网络的转发；

>NOTE: kube-proxy 要求 NODE 节点操作系统中要具备 /sys/module/br_netfilter 文件，而且还要设置 bridge-nf-call-iptables=1，如果不满足要求，那么 kube-proxy 只是将检查信息记录到日志中，kube-proxy 仍然会正常运行，但是这样通过 Kube-proxy 设置的某些 iptables 规则就不会工作。

kube-proxy 有两种实现 service 的方案：userspace 和 iptables：
```txet
1. userspace 是在用户空间监听一个端口，所有的 service 都转发到这个端口，然后 kube-proxy 在内部应用层对其进行转发。因为是在用户空间进行转发，所以效率也不高
2. iptables 完全实现 iptables 来实现 service，是目前默认的方式，也是推荐的方式，效率很高（只有内核中 netfilter 一些损耗）。
```
userspace 只是旧版本支持的模式，以后可能会放弃维护和支持,所以我们目前都是通过 iptables 模式运行 kube-proxy的，后面的分析也是针对这个模式的。
