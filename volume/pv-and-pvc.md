# dynamic
dynamic 的意思也就是说，不需要手工创建pv和pvc，通过配置 storageclass信息，就可以动态创建pv和pvc;

### kubernetes持久化
kubernetes使用两种API资源来管理存储。分别是PersistentVolume（）和PersistentVolumeClaim(PVC)。下面分别介绍下这两种资源的概念。
```
PersistentVolume(简称PV):由管理员设置的存储，它是集群的一部分。就像节点(Node)是集群中的资源一样，PV也是集群中的资源。它包含存储类型，存储大小和访问模式。它的生命周期独立于Pod，例如当使用它的Pod销毁时对PV没有影响。

PersistentVolumeClaim(简称PVC): 是用户存储的请求。它和Pod类似。Pod消耗Node资源，PVC消耗PV资源。Pod可以请求特定级别的资源(CPU和MEM)。PVC可以请求特定大小和访问模式的PV。
```
kubernetes 具备三种方式来访问存储资源：   
**1.直接访问:**
![direct-storage.jpg](/images/direct-storage.jpg)
该种方式移植性比较差，可扩展性差，并且把Volume的基本信息完全暴露给用户，有安全隐患；  
**2.静态PV:**
![static-pv.jpg](/images/static-pv.jpg)
集群管理员创建一些PV。它们带有可供集群用户使用的实际存储的细节。之后便可用于PVC消费。(注意: 这种方式请求的PVC必须要与管理员创建的PV保持一致，如: 存储大小和访问模式)。否则不会将PVC绑定到PV上。  
**3.动态PV:**
![dynamic-pv.jpg](/images/dynamic-pv.jpg)
当管理员创建的静态PV都不匹配用户的PVC,集群可以使用动态的为PVC创建卷。此配置基于StorageClass。PVC请求存储类(StorageClass),并且管理员必须要创建并配置该StorageClass,该StorageClass才能进行动态的创建。
