# pod-anti-affinity
podAffinity 机制是让调度器把一组pods 以某个topologyKey为范围将它们调度到一起， 而 podAntiAffinity 恰好与它相反， 也就是让调度器永远不会将包含 podAntiAffinity 标签的pod 调度到一起，生产环境中，为了满足高可用行，就会非常依赖这个特性， 尽量不要把鸡蛋放到一个篮子里；

#### 应用 podAntiAffinity
创建前端pod, 把前端pod 部署到不同的node上，只需要将 podAffinity 替换成 podAntiAffinity字段名称即可:
```yaml
#frontend.yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 3
  template:
    metadata:
      labels:
        app: frontend
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - topologyKey: kubernetes.io/hostname
            labelSelector:
              matchLabels:
                app: frontend
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
# kubectl  get pod -o wide -l app=frontend
NAME                        READY   STATUS    RESTARTS   AGE   IP            NODE             NOMINATED NODE
frontend-8644dff9b9-4mz6t   0/1     Pending   0          25s   <none>        <none>           <none>
frontend-8644dff9b9-ghgz2   1/1     Running   0          25s   172.30.40.8   192.168.10.242   <none>
frontend-8644dff9b9-zvvhl   1/1     Running   0          25s   172.30.7.4    192.168.10.243   <none>
```
frontend 服务的 pods 被强制性调度到和其它包含label为 "app: backend" pod 所在不同的 node 上了，但是也是为了满足这一要求，有一个pod pending了，因为我们只有两个 node；

#### pod 非亲和性优先级取代强制性要求
与 nodeAffinity 中的 preferredDuringSchedulingIgnoredDuringExecution 一样， 改机制也适用于 podAffinity, 也就是说， 优先将前端 pod 和 后端 pod 调度在相同的 node 上，但是如果不满足需求， 调度到其它 node 也可以接受的。比如:
```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 3
  template:
    metadata:
      labels:
        app: frontend
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 80
            podAffinityTerm:
              topologyKey: kubernetes.io/hostname
              labelSelector:
                matchLabels:
                  app: frontend
      containers:
      - name: main
        image: busybox
        imagePullPolicy: IfNotPresent
        args:
        - sleep
        - "99999"
```
与 pod 亲和性一样， topologyKey 决定了 pod 不能被调度的范围，比如 不能调度到 同一个 tor、 同一个zone、同一个 region等；
