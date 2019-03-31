# 给pod 设置默认的memory requests 和 limits
通过前面的理解， 其实为每个容器显示设置上 requests 和 limits 才是最好的实践。
可以使用 LimitRange 对象来为每个pod 都设置上默认的 requests 和 limits。

#### 创建一个测试用的ns
```bash
$ kubectl create namespace default-mem-example
```

#### 创建一个limitRange:
```yaml
# memory-limitRange-defaults.yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: mem-limit-range
spec:
  limits:
  - default:
      memory: 512Mi
    defaultRequest:
      memory: 256Mi
    type: Container
```
```bash
$ kubectl apply -f  memory-limitRange-defaults.yaml --namespace=constraints-cpu-example
```

#### 在创建一个pod
```yaml
#memory-defaults
apiVersion: v1
kind: Pod
metadata:
  name: default-mem-demo
spec:
  containers:
  - name: default-mem-demo-ctr
    image: nginx
```
```bash
$ kubectl apply -f  memory-defaults.yaml --namespace=constraints-cpu-example
```
#### 指定mem的 limits 不指定 requests 的pod
```yaml
#default-mem-demo-2
apiVersion: v1
kind: Pod
metadata:
  name: default-mem-demo-2
spec:
  containers:
  - name: default-mem-demo-2-ctr
    image: nginx
    resources:
      limits:
        memory: "1Gi"
```
验证效果

```yaml
resources:
  limits:
    memory: 1Gi
  requests:
    memory: 1Gi
```

#### 指定mem的 requests 不指定 limits 的pod
```yaml
# default-mem-demo-3.yaml
apiVersion: v1
kind: Pod
metadata:
  name: default-mem-demo-3
spec:
  containers:
  - name: default-mem-demo-3-ctr
    image: nginx
    resources:
      requests:
        memory: "128Mi"
```

```bash
$ kubectl apply -f  default-mem-demo-3.yaml --namespace=constraints-cpu-example
$ kubectl describe pod default-mem-demo-3 --namespace=constraints-cpu-example
```
查看效果：
```yaml
resources:
  limits:
    memory: 512Mi
  requests:
    memory: 128Mi
```
-------

# 给pod 设置最小最大 memory 资源约束限制
***创建容器时约束验证过程：***
```
1. 如果容器没有指定 requests 和 limits, 将使用默认的资源进行分配
2. 验证容器是否指定了大于或等于min.cpu 的CPU请求限制
3. 验证容器是否指定了小于或等于max.cpu 的CPU请求限制
```
首先为测试环境创建独立的ns
```
kubectl create namespace constraints-mem-example
```
创建一个memory的limitrange对象
```yaml
# memory-constraints.yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: mem-min-max-demo-lr
spec:
  limits:
  - max:
      memory: 1Gi
    min:
      memory: 500Mi
    type: Container
```
创建并验证：
```bash
kubectl apply -f memory-constraints.yaml --namespace=constraints-mem-example
kubectl get limitrange mem-min-max-demo-lr --namespace=constraints-mem-example --output=yaml
```
创建一个超出 memory 约束最大值限制的pod
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: constraints-mem-demo-2
spec:
  containers:
  - name: constraints-mem-demo-2-ctr
    image: nginx
    resources:
      limits:
        memory: "1.5Gi"
      requests:
        memory: "800Mi"
```
创建并验证是否创建成功：
```bash
kubectl apply -f memory-constraints-pod-2.yaml --namespace=constraints-mem-example
```

创建一个小于 memory 约束最小值限制的pod
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: constraints-mem-demo-3
spec:
  containers:
  - name: constraints-mem-demo-3-ctr
    image: nginx
    resources:
      limits:
        memory: "800Mi"
      requests:
        memory: "100Mi"
```
创建并验证：
```bash
kubectl apply -f memory-constraints-pod-3.yaml --namespace=constraints-mem-example
```
最后再创建一个没有指定 memory requests 或 limit限制的pod
```yaml
# memory-constraints-pod-4.yaml
apiVersion: v1
kind: Pod
metadata:
  name: constraints-mem-demo-4
spec:
  containers:
  - name: constraints-mem-demo-4-ctr
    image: nginx
```
```bash
kubectl apply -f memory-constraints-pod-4.yaml --namespace=constraints-mem-example
kubectl get pod constraints-mem-demo-4 --namespace=constraints-mem-example --output=yaml
```

#### 总结：
**limitRange** 应用于单独的pod没有问题， 而我们同时也需要一种手段可以限制命名空间中的可用资源总量。
