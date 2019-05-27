# taint 与 toleration
生产环境中，经常有一些用户提出专机专用的需求，比如业务要给我们提供一些属于他们自己资产的服务器，要加入到我们的 k8s 集群中。
但是他们又同时不想给其它业务共用，以免互相影响，或者他们有一些特权的需求等，总之，希望在 k8s 集群中，给业务划出一个独立小集群；还有一种场景是划分出测试、预发、生产环境；

#### taint & toleration 介绍
为了实现上面的需求，k8s 提供了污点和容忍度机制来实现；
```
taint ：代表污点， 作用在 node 上；
toleration： 代表容忍度， 作用在 pod 上，是否能够容忍 node 上的 taint；
```
放在一起，就表示哪些pod 能容忍哪些 带有taints的 node，没有 toleration 的pod 不能调度到这些node上；

#### 应用 taint & toleration
添加和删除 taint 的语法:
```bash
kubectl taint node [node] key=value[effect]
kubectl taint node [node] key:[effect]-
```
其中[effect] 可取如下值：
```
NoSchedule ：不符合规则的pod，一定不能被调度。
PreferNoSchedule：不符合规则的pod，尽量不要调度。
NoExecute：不仅不会调度，还会驱逐 Node 上已有的且不符合规则的 Pod。
```
**添加和删除 gpu 的 taint:**
```bash
kubectl taint nodes 192.168.10.243 gpu=true:NoSchedule
kubectl taint nodes 192.168.10.243 gpu:NoSchedule-
```

**创建能够容忍 gpu taint 的pod**
```yaml
#bmw-gpu-node-taint.yaml
apiVersion: v1
kind: Pod
metadata:
  name: bmw-gpu-node-taint
spec:
  tolerations: #设置容忍性
  - key: "gpu"
    operator: "Equal"  #如果操作符为Exists，那么value属性可省略,如果不指定operator，则默认为Equal
    value: "true"
    effect: "NoSchedule"
  containers:
  - image: bmwx4/kugo:v1.0
    imagePullPolicy: IfNotPresent
    name: kugo
```

**创建并验证**
```bash
# kubectl  get pod -o wide
bmw-gpu-node-taint                1/1     Running   0          17s     172.30.7.7    192.168.10.243   <none>
```

>PS: toleration 存在两种特殊情况：
如果一个 toleration 的 key 为空且 operator 为 Exists ，表示这个 toleration 与任意的 key 、 value 和 effect 都匹配，即这个 toleration 能容忍任意 taint。
```yaml
tolerations:
- operator: "Exists"
```
如果一个 toleration 的 effect 为空，则 key 值与之相同的相匹配 taint 的 effect 可以是任意值。
```yaml
tolerations:
- key: "key"
  operator: "Exists"
```

#### 配置 Node 失效之后的pod重新调度最长等待时间
如果给一个 Node 添加了一个 effect 值为 NoExecute 的 taint，则任何不能忍受这个 taint 的 pod 都会马上被驱逐；任何可以容忍这个 taint 的 pod 都不会被驱逐。但是，如果 pod 存在一个 effect 值为 NoExecute 的 toleration 指定了可选属性 tolerationSeconds 的值，则表示在给节点添加了上述与其匹配的 taint 之后，pod 还能继续在节点上运行的时间。例如，
```yaml
tolerations:
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoExecute"
  tolerationSeconds: 3600
```

这表示如果这个 pod 正在运行，然后一个匹配的 taint 与被添加到该pod所在node的taint相同，那么 pod 还将继续在节点上运行 3600 秒，然后被驱逐。如果在此之前上述 taint 被删除了，则 pod 不会被驱逐。


>PS: 基于 污点信息 驱逐 pod 的功能不是默认开启的， 如果要启用这个特性，需要运行 kube-controller-manager时，使用 --feature-gates=TaintBasedEvictions=true 选项;
