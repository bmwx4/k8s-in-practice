# nodeAffinity
node affinity 也叫节点亲和性的意思， 这种机制允许你通知 kube-scheduler 只调度到某几个节点子集上面；
与node selector 类似，节点必须包含 pod 所对应字段中的指定label，才能成为 pod 调度的目标节点。

#### 检查节点上的默认标签
```
# kubectl  get node --show-labels                           
NAME             STATUS   ROLES    AGE   VERSION   LABELS
192.168.10.242   Ready    <none>   69d   v1.11.6   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=192.168.10.242
192.168.10.243   Ready    <none>   55d   v1.11.6   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=192.168.10.243
```

#### 指定强制性节点亲和性规则
假如有一组pod需要调度到 gpu 的 node 上，如何设置？  
使用 nodeSelector 实现:
```yaml
# bmw-gpu-node-selector.yaml
apiVersion: v1
kind: Pod
metadata:
  name: bmw-gpu-node-selector
spec:
  nodeSelector:
    gpu: "true"
  containers:
  - image: bmwx4/kugo:v1.0
    imagePullPolicy: IfNotPresent
    name: kugo
```
使用 nodeAffinity 实现:
```yaml
# bmw-gpu-node-affinity.yaml
apiVersion: v1
kind: Pod
metadata:
  name: bmw-gpu-node-affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: gpu
            operator: In
            values:
            - "true"
  containers:
  - image: bmwx4/kugo:v1.0
    imagePullPolicy: IfNotPresent
    name: kugo
```
创建并验证:
```bash
# kubectl  create -f bmw-gpu-node-affinity.yaml
# kubectl  get pod -o wide
NAME                     READY   STATUS    RESTARTS   AGE   IP            NODE             NOMINATED NODE
bmw-gpu-node-affinity    0/1     Pending   0          10s   <none>        <none>           <none>
```
因为我们没有给node 设置node label 导致的无法调度，下面给node02 设置上gpu的 node label:
```bash
# kubectl label nodes 192.168.10.243 gpu=true
node/192.168.10.243 labeled
# kubectl  get pod -o wide                                   
NAME                     READY   STATUS    RESTARTS   AGE    IP            NODE             NOMINATED NODE
bmw-gpu-node-affinity    1/1     Running   0          2m2s   172.30.7.3    192.168.10.243   <none>
```
**nodeAffinity 字段的理解**
requiredDuringSchedulingIgnoredDuringExecution 包含两部分含义:
```
requiredDuringScheduling...: 表示该字段下定义的规则，明确表示出 node 上必须包含的 node label；
...IgnoredDuringExecution:  表示该字段下定义的规则，不会影响已经存在node上运行着的pod。
```
***PS:***
> 当前的亲和性规则只会影响正在被调度的 pod，并且不会导致已经在运行的pod被驱逐，这就是为什么目前的规则都是以 IgnoredDuringExecution 结尾的，以后，会不会加入 requiredDuringExecution 也说不定；
这里的匹配逻辑是 label 的值在某个列表中，可选的操作符有：

**operator可选的值**  
```
In：label 的值在某个列表中
NotIn：label 的值不在某个列表中
Exists：某个 label 存在
DoesNotExist: 某个 label 不存在
Gt：label 的值大于某个值（字符串比较）
Lt：label 的值小于某个值（字符串比较）
```

nodeSelectorTerms 和 matchExpressions 字段定义了节点的标签，必须满足哪一种表达式，才能满足 pod 的调度条件，比如上面的例子可以理解成，node 必须包含 一个叫做 gpu的标签，并且标签的值为 true。选择过程如下图：
![affinity](/images/node-affinity.png)

#### 调度pod时， 优先考虑某些节点
当调度某个pod的到gpu集群上的时候，  当gpu集群的资源不足为这些pod分配了，或者出于其他原因，不能被调度上这些node的时候，用户可以指定调度器优先考虑这些node，但是，当出现上面的情况时，调度到其他node上，也是可以接受的，比如:
```yaml
# bmw-gpu-node-affinity-preferred.yaml
apiVersion: v1
kind: Pod
metadata:
  name: bmw-gpu-node-affinity-preferred
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: gpu
            operator: In
            values:
            - "true"
  containers:
  - image: bmwx4/kugo:v1.0
    imagePullPolicy: IfNotPresent
    name: bmwx4
```
创建并验证:
```bash
# kubectl  get node --show-labels
NAME             STATUS   ROLES    AGE   VERSION   LABELS
192.168.10.242   Ready    <none>   69d   v1.11.6   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=192.168.10.242
192.168.10.243   Ready    <none>   55d   v1.11.6   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=192.168.10.243
# kubectl  create -f bmw-gpu-node-affinity-preferred.yaml
# kubectl  get pod
NAME                              READY   STATUS    RESTARTS   AGE
bmw-gpu-node-affinity-preferred   1/1     Running   0          3s
```
**weight 权重**  
因为集群中肯定有多重被划分的小集群，如果需要给pod分有多个节点亲和性规则，那么根据 weight 来进行节点优先级划分。

[Node affinity and NodeSelector 设计文档](
https://github.com/kubernetes/community/blob/master/contributors/design-proposals/nodeaffinity.md)
