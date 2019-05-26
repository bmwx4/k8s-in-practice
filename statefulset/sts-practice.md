# StatefulSet Practice

#### 创建 StatefulSet
```yaml
# web.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None  # 设置为Node，代表是 Headless Service
  selector:
    app: nginx
---
apiVersion: apps/v1beta1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx"
  replicas: 5
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80
          name: web
```
创建并验证结果:
```bash
# kubectl  create  -f web.yaml
# kubectl  get sts -o wide
```

#### StatefulSet 部署和伸缩时与 Deployment 的区别
StatefulSet中反复强调的“稳定的网络标识”，主要指Pods的hostname以及对应的DNS Records。

#### StatefulSet 部署和伸缩时与 Deployment 的区别
```
1. 当部署有N个副本的StatefulSet应用时，严格按照index从0到N-1的递增顺序创建，下一个Pod创建必须是前一个Pod Ready为前提。
2. 当删除有N个副本的StatefulSet应用时，严格按照index从N-1到0的递减顺序删除，下一个Pod删除必须是前一个Pod shutdown并完全删除为前提。
3. 当扩容StatefulSet应用时，每新增一个Pod必须是前一个Pod Ready为前提。
4. 当缩容StatefulSet应用时，没删除一个Pod必须是前一个Pod shutdown并成功删除为前提。
5. 注意StatefulSet的pod.Spec.TerminationGracePeriodSeconds不要设置为0。
```

扩缩容：
```bash
kubectl scale sts web --replicas=10
kubectl scale sts web --replicas=5
kubectl scale sts web --replicas=0
```

#### sts 应用ceph-rbd
使用sts 可以应用动态创建pv模式，只需要配置 storageClassName 和 PVC 的名称前缀即可；
```yaml
# sts-with-rbd.yaml
apiVersion: apps/v1beta1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx"
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: "dynamic-ceph-rdb"
      resources:
        requests:
          storage: 50Mi      
```
执行创建并验证:
```bash
# kubectl  create -f dynamic.yaml
# kubectl  get pvc
NAME            STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS       AGE
www-web-0   Bound    pvc-64aba118-79d2-11e9-8bb5-000c29a5444f   50Mi       RWO            dynamic-ceph-rdb   23s
www-web-1   Bound    pvc-6e0483ae-79d2-11e9-8bb5-000c29a5444f   50Mi       RWO            dynamic-ceph-rdb   8s
# kubectl  get pod -l app=nginx -o wide
NAME        READY   STATUS    RESTARTS   AGE    IP           NODE             NOMINATED NODE
web--0   1/1     Running   0          116s   172.30.7.7   192.168.10.243   <none>
web--1   1/1     Running   0          101s   172.30.7.9   192.168.10.243   <none>
```

#### 网络节点失效
当 Node Condition 是 NetworkUnavailable 时，node controller不会强制从 apiserver 中删除这个Node上的这些pods对象，这些pods的state在apiserver中被标记为Terminating或者Unknown，因此StatefulSet Controller并不会在其他Node上再recreate同identity的Pods。当你确定了这个Node上的StatefulSet Pods shutdown或者无法和该StatefulSet的其他Pods网络通信时，你就需要强制删除 apiserver 中这些unreachable pods 了，然后 StatefulSet Controller 就能在其他Ready Nodes上recreate同  identity 的Pods，使得 StatefulSet 继续健康工作。  
节点失效测试：
```
# ifconfig enss33 down
# kubectl get node
# kubectl get pod
```
