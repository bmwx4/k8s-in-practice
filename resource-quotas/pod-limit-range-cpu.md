# 给pod 设置默认的CPU资源约束限制
本节介绍如何设置 cpu 的默认资源限制，以及最小、最大CPU资源限制

#### 创建一个测试用的ns
```bash
$ kubectl create namespace default-cpu-example
```
为pod创建一个 limitRange 对象
```yaml
cat << EOF > cpu-defaults.yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: cpu-limit-range
spec:
  limits:
  - default:
      cpu: 1
    defaultRequest:
      cpu: 0.5
    type: Container
EOF    
```
创建并验证
```bash
kubectl apply -f cpu-defaults.yaml --namespace=default-cpu-example
```
创建一个pod,用来验证上面limitrange：
```yaml
cat << EOF > cpu-defaults-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: default-cpu-demo
spec:
  containers:
  - name: default-cpu-demo-ctr
    image: nginx
    imagePullPolicy: IfNotPresent
EOF    
```
创建并验证：
```bash
kubectl apply -f cpu-defaults-pod.yaml --namespace=default-cpu-example
kubectl get pod default-cpu-demo --output=yaml --namespace=default-cpu-example
```
再创建一个不指定 requests,但是指定limit的 pod
```yaml
cat << EOF >  cpu-defaults-pod-2.yaml
apiVersion: v1
kind: Pod
metadata:
  name: default-cpu-demo-2
spec:
  containers:
  - name: default-cpu-demo-2-ctr
    image: nginx
    imagePullPolicy: IfNotPresent
    resources:
      limits:
        cpu: "2"
EOF        
```
创建并验证效果：
```bash
kubectl apply -f cpu-defaults-pod-2.yaml --namespace=default-cpu-example
kubectl get pod default-cpu-demo-2 --output=yaml --namespace=default-cpu-example
```
***PS:***
>requests 并没有使用default requests

再创建一个不指定 limit ,但是指定 requests 的pod
```yaml
cat << EOF > cpu-defaults-pod-3.yaml
apiVersion: v1
kind: Pod
metadata:
  name: default-cpu-demo-3
spec:
  containers:
  - name: default-cpu-demo-3-ctr
    image: nginx
    imagePullPolicy: IfNotPresent    
    resources:
      requests:
        cpu: "0.75"
EOF        
```
创建并验证：
```bash
kubectl apply -f cpu-defaults-pod-3.yaml --namespace=default-cpu-example
kubectl get pod default-cpu-demo-3 --output=yaml --namespace=default-cpu-example
```
***PS:***
>limits 使用了default limits
-----
请确保演示功能已经实现，清除演示环境的命名空间 
```bash
kubectl delete namespace default-cpu-example
```

# 给pod 设置最大和最小的CPU资源约束限制

***创建容器时约束验证过程：***
```
1. 如果容器没有指定 requests 和 limits, 将使用默认的资源进行分配
2. 验证容器是否指定了大于或等于min.cpu 的CPU请求限制
3. 验证容器是否指定了小于或等于max.cpu 的CPU请求限制
```

#### 创建一个测试用的ns
```bash
$ kubectl create namespace constraints-cpu-example
```

创建一个最小和最大约束的 LimitRange
```yaml
cat << EOF >  cpu-min-max-demo-lr.yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: cpu-min-max-demo-lr
spec:
  limits:
  - max:
      cpu: "800m"
    min:
      cpu: "200m"
    type: Container
EOF

```
这个配置表示，创建的CPU资源不允许超过800m，但至少要满足200m;
创建并查看：
```bash
kubectl apply -f  cpu-min-max-demo-lr.yaml --namespace=constraints-cpu-example
kubectl get limitrange cpu-min-max-demo-lr --output=yaml --namespace=constraints-cpu-example
```


下面创建一个合法的pod验证一下限制约束是否生效？
```yaml
cat <<EOF > cpu-constraints-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: constraints-cpu-demo
spec:
  containers:
  - name: constraints-cpu-demo-ctr
    image: nginx
    imagePullPolicy: IfNotPresent
    resources:
      limits:
        cpu: "800m"
      requests:
        cpu: "500m"
EOF        
```

验证是否可以创建成功：
```bash
kubectl apply -f cpu-constraints-pod.yaml --namespace=constraints-cpu-example
kubectl get pod constraints-cpu-demo --namespace=constraints-cpu-example
```

下面再创建一个超出最大约束条件的pod验证一下限制约束是否生效？
```yaml
cat << EOF > cpu-constraints-pod-2.yaml
apiVersion: v1
kind: Pod
metadata:
  name: constraints-cpu-demo-2
spec:
  containers:
  - name: constraints-cpu-demo-2-ctr
    image: nginx
    imagePullPolicy: IfNotPresent    
    resources:
      limits:
        cpu: "1.5"
      requests:
        cpu: "500m"
EOF        
```
验证是否可以创建成功：
```bash
kubectl apply -f cpu-constraints-pod-2.yaml --namespace=constraints-cpu-example
kubectl get pod constraints-cpu-demo2 --namespace=constraints-cpu-example
```

下面再创建一个没有满足最小约束条件的pod验证一下限制约束是否生效？
```yaml
cat << EOF  > cpu-constraints-pod-3.yaml
apiVersion: v1
kind: Pod
metadata:
  name: constraints-cpu-demo-3
spec:
  containers:
  - name: constraints-cpu-demo-3-ctr
    image: nginx
    imagePullPolicy: IfNotPresent    
    resources:
      limits:
        cpu: "800m"
      requests:
        cpu: "100m"
EOF        
```
验证是否可以创建成功：
```bash
kubectl apply -f cpu-constraints-pod-3.yaml --namespace=constraints-cpu-example
kubectl get pod constraints-cpu-demo3 --namespace=constraints-cpu-example
```

下面创建一个没有指定CPU requests 和 limits的pod
```yaml
cat << EOF > cpu-constraints-pod-4.yaml
apiVersion: v1
kind: Pod
metadata:
  name: constraints-cpu-demo-4
spec:
  containers:
  - name: constraints-cpu-demo-4-ctr
    image: ubuntu-stress
    imagePullPolicy: IfNotPresent
EOF    
```
验证是否可以创建成功：
```bash
kubectl apply -f cpu-constraints-pod-4.yaml --namespace=constraints-cpu-example
kubectl get pod constraints-cpu-demo4 --namespace=constraints-cpu-example
```
请确保演示功能已经实现，清理测试环境：
```bash
kubectl delete namespace constraints-cpu-example
```
