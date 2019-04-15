# 限制可创建对象的个数
资源配额同样可以限制单个命名空间中的pod、ReplicationController、Service 以及其他对象的个数。集群管理员可以根据一些策略限制用户能够创建的对象个数，也可以用来限制公网IP或者Service可使用的端口个数；

创建一个ns进行验证：
```bash
$ kubectl create namespace quota-pod-example
```
下面是 ResourceQuota 对象信息：
```yaml
cat << EOF >  quota-pod.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: pod-demo
spec:
  hard:
    pods: "2"
EOF

```
创建并查看结果：
```bash
kubectl create -f quota-pod.yaml --namespace=quota-pod-example
kubectl get resourcequota pod-demo --namespace=quota-pod-example --output=yaml
# ResourceQuota 的作用对象是在 命名空间级( namespace )
#   --namespace 参数是指定该属性作用于那个命名空间。默认是 default 空间。
```
下面创建一个 Deployment对象，副本数为3,用来触发podCount限制：
```yaml
cat  <<EOF >  quota-pod-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pod-quota-demo
spec:
  selector:
    matchLabels:
      purpose: quota-demo
  replicas: 3
  template:
    metadata:
      labels:
        purpose: quota-demo
    spec:
      containers:
      - name: pod-quota-demo
        image: nginx
		imagePullPolicy: IfNotPresent
EOF	
```
```bash
kubectl apply -f quota-pod-deployment.yaml --namespace=quota-pod-example
# 查看命名空间为 quota-pod-example 下的 pod 状态
kubectl get pod -n quota-pod-example -w 
# 退出watch 状态，按住 ctrl + C 
# 查看该资源对象的状态
kubectl describe -f quota-pod-deployment.yaml
```

#查看命令
kubectl describe deployments -n quota-pod-example

还可以限制哪些API Object 呢？

```yaml
spec:
  hard:
    persistentvolumeclaims: "1"
    services.loadbalancers: "2"
    services.nodeports: "0"
    ....
```

查看k8s源代码，可以发现如下定义：
文件路径： pkg/apis/core/types.go
```golang
// The following identify resource constants for Kubernetes object types
const (
	// Pods, number
	ResourcePods ResourceName = "pods"
	// Services, number
	ResourceServices ResourceName = "services"
	// ReplicationControllers, number
	ResourceReplicationControllers ResourceName = "replicationcontrollers"
	// ResourceQuotas, number
	ResourceQuotas ResourceName = "resourcequotas"
	// ResourceSecrets, number
	ResourceSecrets ResourceName = "secrets"
	// ResourceConfigMaps, number
	ResourceConfigMaps ResourceName = "configmaps"
	// ResourcePersistentVolumeClaims, number
	ResourcePersistentVolumeClaims ResourceName = "persistentvolumeclaims"
	// ResourceServicesNodePorts, number
	ResourceServicesNodePorts ResourceName = "services.nodeports"
	// ResourceServicesLoadBalancers, number
	ResourceServicesLoadBalancers ResourceName = "services.loadbalancers"
	// CPU request, in cores. (500m = .5 cores)
	ResourceRequestsCPU ResourceName = "requests.cpu"
	// Memory request, in bytes. (500Gi = 500GiB = 500 * 1024 * 1024 * 1024)
	ResourceRequestsMemory ResourceName = "requests.memory"
	// Storage request, in bytes
	ResourceRequestsStorage ResourceName = "requests.storage"
	// Local ephemeral storage request, in bytes. (500Gi = 500GiB = 500 * 1024 * 1024 * 1024)
	ResourceRequestsEphemeralStorage ResourceName = "requests.ephemeral-storage"
	// CPU limit, in cores. (500m = .5 cores)
	ResourceLimitsCPU ResourceName = "limits.cpu"
	// Memory limit, in bytes. (500Gi = 500GiB = 500 * 1024 * 1024 * 1024)
	ResourceLimitsMemory ResourceName = "limits.memory"
	// Local ephemeral storage limit, in bytes. (500Gi = 500GiB = 500 * 1024 * 1024 * 1024)
	ResourceLimitsEphemeralStorage ResourceName = "limits.ephemeral-storage"
)

#### 清理
```bash
kubectl delete namespace quota-pod-example
```

#### 扩展
***为特定状态或者QoS等级的pod 设置配额 ***
上面的测试中，ResourceQuota 都作用在了所有的pod等对象，但是这样相对来说限制的策略还是比较粗放的，k8s 也给出除了 quota scopes, 让限制粒度更加细化；

```go
Scopes []ResourceQuotaScope

// A ResourceQuotaScope defines a filter that must match each object tracked by a quota
type ResourceQuotaScope string
const (
	// Match all pod objects where spec.activeDeadlineSeconds
	ResourceQuotaScopeTerminating ResourceQuotaScope = "Terminating"
	// Match all pod objects where !spec.activeDeadlineSeconds
	ResourceQuotaScopeNotTerminating ResourceQuotaScope = "NotTerminating"
	// Match all pod objects that have best effort quality of service
	ResourceQuotaScopeBestEffort ResourceQuotaScope = "BestEffort"
	// Match all pod objects that do not have best effort quality of service
	ResourceQuotaScopeNotBestEffort ResourceQuotaScope = "NotBestEffort"
	// Match all pod objects that have priority class mentioned
	ResourceQuotaScopePriorityClass ResourceQuotaScope = "PriorityClass"
)
```
BestEffort 与 NotBestEffort 相对，就是指是否满足 BestEffort QoS级别的pods。
Terminating 与 NotTerminating 相对，因为用户可以在pod spec中配置 activeDeadlineSeconds 字段来标志失败 (Failed) Pod的重试最大时间，超过这个时间不会继续重试，常用于Job。Terminating 配额的作用是应用于配置了这个字段的pod，而 NotTerminating 则应用于没有配置该字段的pod。

另外：配额范围的取值，也影响着可以限制的内容，比如 BestEffort 只限制pod数量；

比如:
```yaml

apiVersion: v1
kind: ResourceQuota
metadata:
  name: pod-demo
spec:
  scopes:
  - BestEffort
  - NotTerminating
  hard:
    pods: "2"
```
意思就是说，这个quota 只会应用于拥有 BestEffort QoS, 以及没有设置有效期的pod上，这样的pod只允许存在4个。
