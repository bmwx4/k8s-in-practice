# kubernetes中的对象

在 Kubernetes 集群中，***Kubernetes 对象*** 是持久化实体。Kubernetes 使用这些对象实体去表示整个集群的状态。比如：

- 什么容器化应用在运行（以及在哪个 Node 上）
- 可以被应用使用的资源
- 关于应用如何进行状态变更的策略，比如重启策略、升级策略，以及容错策略

#### 对象的理解

```go
type Object struct {
	metav1.TypeMeta `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty" protobuf:"bytes,1,opt,name=metadata"`
	Spec ObjectSpec `json:"spec,omitempty" protobuf:"bytes,2,opt,name=spec"`
	Status ObjectStatus `json:"status,omitempty" protobuf:"bytes,3,opt,name=status"`
}
//k8s.io/apimachinery/pkg/apis/meta/v1/types.go
// TypeMeta describes an individual object in an API response or request
// with strings representing the type of the object and its API schema version.
// Structures that are versioned or persisted should inline TypeMeta.
type TypeMeta struct {
	// Kind is a string value representing the REST resource this object represents.
	Kind string `json:"kind,omitempty" protobuf:"bytes,1,opt,name=kind"`
	// APIVersion defines the versioned schema of this representation of an object.
	APIVersion string `json:"apiVersion,omitempty" protobuf:"bytes,2,opt,name=apiVersion"`
}
// ObjectMeta is metadata that all persisted resources must have, which includes all objects users must create.
type ObjectMeta struct {
	// Name must be unique within a namespace. Is required when creating resources, although
	Name string `json:"name,omitempty" protobuf:"bytes,1,opt,name=name"`
	GenerateName string `json:"generateName,omitempty" protobuf:"bytes,2,opt,name=generateName"`
	Namespace string `json:"namespace,omitempty" protobuf:"bytes,3,opt,name=namespace"`
	SelfLink string `json:"selfLink,omitempty" protobuf:"bytes,4,opt,name=selfLink"`
	UID types.UID `json:"uid,omitempty" protobuf:"bytes,5,opt,name=uid,casttype=k8s.io/kubernetes/pkg/types.UID"`
	ResourceVersion string `json:"resourceVersion,omitempty" protobuf:"bytes,6,opt,name=resourceVersion"`
	Generation int64 `json:"generation,omitempty" protobuf:"varint,7,opt,name=generation"`
	CreationTimestamp Time `json:"creationTimestamp,omitempty" protobuf:"bytes,8,opt,name=creationTimestamp"`
	DeletionTimestamp *Time `json:"deletionTimestamp,omitempty" protobuf:"bytes,9,opt,name=deletionTimestamp"`
	DeletionGracePeriodSeconds *int64 `json:"deletionGracePeriodSeconds,omitempty" protobuf:"varint,10,opt,name=deletionGracePeriodSeconds"`
	Labels map[string]string `json:"labels,omitempty" protobuf:"bytes,11,rep,name=labels"`
	Annotations map[string]string `json:"annotations,omitempty" protobuf:"bytes,12,rep,name=annotations"`
	OwnerReferences []OwnerReference `json:"ownerReferences,omitempty" patchStrategy:"merge" patchMergeKey:"uid" protobuf:"bytes,13,rep,name=ownerReferences"`
	Initializers *Initializers `json:"initializers,omitempty" protobuf:"bytes,16,opt,name=initializers"`
	Finalizers []string `json:"finalizers,omitempty" patchStrategy:"merge" protobuf:"bytes,14,rep,name=finalizers"`
	ClusterName string `json:"clusterName,omitempty" protobuf:"bytes,15,opt,name=clusterName"`
}
```

- `apiVersion` - 创建该对象所使用的 Kubernetes API 的版本
- `kind` - 想要创建的对象的类型
- `metadata` - 帮助识别对象唯一性的数据,比如name，uuid等
- `spec` 它描述了对象的 *期望状态*， 表示希望对象所具有的特征。
- `status` 描述了对象的 *实际状态*，它是由 Kubernetes 系统提供和更新。在任何时刻，Kubernetes 的控制器都会管理着对象的实际状态以与我们所期望的状态相匹配。

对象 `spec` 的精确格式对每个 Kubernetes 对象来说是不同的，包含了特定于该对象的嵌套字段。[Kubernetes API 参考](https://kubernetes.io/docs/api/)能够帮助我们找到任何我们想创建的对象的 spec 格式。

-------
#### Kubernetes 对象分类：

| 类别 | 名称 |
| :----------: | :-------- |
| 资源对象 | Pod、ReplicaSet、ReplicationController、Deployment、StatefulSet、DaemonSet、Job、CronJob、HorizontalPodAutoscaling、Node、Namespace、Service、Ingress、Label、CustomResourceDefinition |
| 存储对象 | Volume、PersistentVolume、Secret、ConfigMap                  |
| 策略对象 | SecurityContext、ResourceQuota、LimitRange                   |
| 身份对象 | ServiceAccount、Role、ClusterRole                            |

#### 举例说明 Kubernetes 对象
当创建 Kubernetes 对象时，必须提供对象的 spec，用来描述该对象的期望状态，以及关于对象的一些基本信息（例如，名称）。当使用 Kubernetes API 创建对象时（或者直接创建，或者基于`kubectl`），API 请求必须在请求体中包含 JSON 格式的信息。**更常用的是，需要在 .yaml 文件中为 kubectl 提供这些信息**。 `kubectl` 在执行 API 请求时，将这些信息转换成 JSON 格式。

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80
```

```bash
$ kubectl create -f nginx-deployment.yaml --record
deployment.extensions/nginx-deployment created
```

关于对象 spec、status 和 metadata 更多信息，查看 [Kubernetes API Conventions]( https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md)。
