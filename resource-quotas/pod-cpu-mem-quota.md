# 如何限制命名空间中的可用资源总量 ResourceQuota

我们可以使用 ResourceQuota 对象来检查将要创建的pod 是否会引起总资源量的使用超出限额，如果超出限额， 创建请求会被拒绝。

ResourceQuota 在代码中是如何定义的?
```go
// pkg/apis/core/types.go
// ResourceQuota sets aggregate quota restrictions enforced per namespace
type ResourceQuota struct {
	metav1.TypeMeta
	// +optional
	metav1.ObjectMeta

	// Spec defines the desired quota
	// +optional
	Spec ResourceQuotaSpec

	// Status defines the actual enforced quota and its current usage
	// +optional
	Status ResourceQuotaStatus
}

// ResourceQuotaSpec defines the desired hard limits to enforce for Quota
type ResourceQuotaSpec struct {
	// Hard is the set of desired hard limits for each named resource
	// +optional
	Hard ResourceList
	// A collection of filters that must match each object tracked by a quota.
	// If not specified, the quota matches all objects.
	// +optional
	Scopes []ResourceQuotaScope
	// ScopeSelector is also a collection of filters like Scopes that must match each object tracked by a quota
	// but expressed using ScopeSelectorOperator in combination with possible values.
	// +optional
	ScopeSelector *ScopeSelector
}
```

*** ResourceQuota 何时生效？***
因为资源配额在pod创建的时候进行检查， 所以 ResourceQuota 对象仅仅作用于在其后创建的pod --- 并不影响已经存在的pod。

#### 如何使用 ResourceQuota
为测试环境创建一个ns
```bash
kubectl create namespace quota-mem-cpu-example
```

创建 ResourceQuota 对象
```yaml
cat << EOF >  mem-cpu-demo.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: mem-cpu-demo
spec:
  hard:
    requests.cpu: "1"
    requests.memory: 1Gi
    limits.cpu: "2"
    limits.memory: 2Gi
EOF    
```
创建并验证 ResourceQuota 对象
```bash
$ kubectl create namespace quota-mem-cpu-example
$ kubectl create -f mem-cpu-demo.yaml --namespace=quota-mem-cpu-example
```

创建一个符合quota限制的pod
```yaml
cat << EOF > quota-mem-cpu-demo.yaml
apiVersion: v1
kind: Pod
metadata:
  name: quota-mem-cpu-demo
spec:
  containers:
  - name: quota-mem-cpu-demo-ctr
    image: nginx
    resources:
      limits:
        memory: "800Mi"
        cpu: "800m"
      requests:
        memory: "600Mi"
        cpu: "400m"
EOF
```
创建并验证：
```bash
$ kubectl create -f quota-mem-cpu-demo.yaml --namespace=quota-mem-cpu-example
```
然后 查看当前 resourcequota 使用情况：
```bash
$ kubectl get resourcequota mem-cpu-demo --namespace=quota-mem-cpu-example --output=yaml
status:
  hard:
    limits.cpu: "2"
    limits.memory: 2Gi
    requests.cpu: "1"
    requests.memory: 1Gi
  used:
    limits.cpu: 800m
    limits.memory: 800Mi
    requests.cpu: 400m
    requests.memory: 600Mi
```

再创建第二个pod，尝试突破 quota 限制
```yaml
cat <<EOF > quota-mem-cpu-demo-2.yaml
apiVersion: v1
kind: Pod
metadata:
  name: quota-mem-cpu-demo-2
spec:
  containers:
  - name: quota-mem-cpu-demo-2-ctr
    image: redis
	imagePullPolicy: IfNotPresent
    resources:
      limits:
        memory: "1Gi"
        cpu: "800m"      
      requests:
        memory: "700Mi"
        cpu: "400m"
EOF	
```
创建并验证：
```bash
$ kubectl create -f quota-mem-cpu-demo2.yaml --namespace=quota-mem-cpu-example
$ kubectl get quota-mem-cpu-demo-2 -n quota-mem-cpu-example
```

***思考：***
如果指创建了quota，在没有limitrange的情况下，并且不指定 resources， 是否可以创建 pod 呢？

***结论：***
pod将无法创建成功， 必须为这些对象指定 resources 才可以， 否则API 不会接收pod的创建请求。

#### 总结:
LimitRange 应用于单独的pod； ResourceQuota 应用于命名空间中所有的pod；
![limitRange-resourcequota](../images/limitRange-vs-resourcequota.png)

#### 清理环境
```bash
kubectl delete namespace quota-mem-cpu-example
```
