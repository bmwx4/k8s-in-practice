# pod-affinity
nodeAffinity 机制只是影响了 pod 与 node 之间的亲和性，但是，在生产环境里， 需要通过 podaffinity 指定pod自身之间的亲和性，比如 前端pod和后端pod，部署的近可以降低延时；同时，可以使用 podAffinity 机制将一组pods调度到 同一个node、同一个Tor、同一个Zone 等；

#### 应用 podAffinity
比如一个后端pod和一组前端pod ，要求部署到同一个node上；首先部署一个后端pod:
```bash
kubectl run backend -l app=backend --image=busybox --image-pull-policy=IfNotPresent -- sleep 1h
```
创建前端pod:
```yaml
#frontend.yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 2
  template:
    metadata:
      labels:
        app: frontend
    spec:
      affinity:
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - topologyKey: kubernetes.io/hostname
            labelSelector:
              matchLabels:
                app: backend
      containers:
      - name: main
        image: busybox
        imagePullPolicy: IfNotPresent
        args:
        - sleep
        - "99999"
```
创建并验证效果:
```bash
# kubectl  get pod -o wide
NAME                              READY   STATUS    RESTARTS   AGE     IP            NODE             NOMINATED NODE
backend-7459854fff-fmsb9          1/1     Running   0          4m11s   172.30.7.5    192.168.10.243   <none>
frontend-859f886dfc-qb5vw         1/1     Running   0          4s      172.30.7.6    192.168.10.243   <none>
frontend-859f886dfc-rvdxs         1/1     Running   0          4s      172.30.7.4    192.168.10.243   <none>
```
frontend 服务被强制性调度到和其它包含label为 "app: backend" pod所在的node上, 关键字段解释:
```
podAffinity: 定义 pod 亲和性
requiredDuringSchedulingIgnoredDuringExecution: 定义一个强制性需求
topologyKey: 本次部署的pod ，必须被调度到匹配 pod 选择器的node上
```
调度原理如下图:
![podaffinity](/images/pod-affinity.png)

#### 将 pod 调度到同一个tor、zone、region
在前面的例子里，说了可以通过 podaffinity 机制将pod部署在同一node上，使用相同方案，通过修改 topologyKey 字段，将pod 调度到更多其它维度层面上。
比如：  
**部署在相同的tor**  
给每个node打上实际的 tor标签,因为一个tor，对应多台node
```bash
kubectl  label  node 192.168.10.243 tor=xxx
```
然后podaffinity 的描述：
```yaml
podAffinity:
  requiredDuringSchedulingIgnoredDuringExecution:
  - topologyKey: tor
    labelSelector:
      matchLabels:
        app: backend
```
**部署在相同的zone**  
给每个node打上实际的 zone 标签
```bash
kubectl  label  node 192.168.10.243 zone=xxx
```
然后podaffinity 的描述：
```yaml
podAffinity:
  requiredDuringSchedulingIgnoredDuringExecution:
  - topologyKey: zone
    labelSelector:
      matchLabels:
        app: backend
```
部署在相同的region与上面的调度时同理；
>一句话解释: topologyKey 决定了 pod 被调度的范围

#### pod 亲和性优先级取代强制性要求
与 nodeAffinity 中的 preferredDuringSchedulingIgnoredDuringExecution 一样， 改机制也适用于 podAffinity, 也就是说， 优先将前端 pod 和 后端 pod 调度在相同的 node 上，但是如果不满足需求， 调度到其它 node 也可以接受的。比如:
```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 2
  template:
    metadata:
      labels:
        app: frontend
    spec:
      affinity:
        podAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 80
            podAffinityTerm:
              topologyKey: kubernetes.io/hostname
              labelSelector:
                matchLabels:
                  app: backend
      containers:
      - name: main
        image: busybox
        imagePullPolicy: IfNotPresent
        args:
        - sleep
        - "99999"
```
与 nodeAffinity 优先级规则一样，需要为一个规则设置一个权重。同时也需要  topologyKey 和 labelSelector。与 required 原理相比， preferred 调度原理如下图:
![pod-affinity-preferred](/images/pod-affinity-preferred.png)
