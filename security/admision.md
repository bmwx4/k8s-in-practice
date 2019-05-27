# 服务账户准入控制器 Admission Controller
#### Admission Controller Background:
k8s 已经提供了  authentication/authorization，而且前面我们已经了解RBAC的用法了，那为什么 k8s 还会引入 admission 这种机制？
```
1. authentication/authorization 是 Kubernetes 的认证鉴权，运行在 filter 中，只能获取 http 请求 header 以及证书，并不能获取请求的 body。所以 authn/authz 只能对客户端进行认证和鉴权，不可以对请求的对象进行任何操作，因为这里根本还获取不到对象。
2. Admission 运行在 API Server 的增删改查 handler 中，在对对象被持久化之前，拦截对 API Server的请求，它可以自然地操作 API resource。
```

#### Admission Controller Workflow:
API Server 接收到客户端请求后首先进行认证和鉴权，认证鉴权通过后才会进行后续的endpoint handler处理。
![workflow](/images/admission-controller-phases.png)
具体流程可以理解如下:
```
1. 当API Server 接收到对象后,根据 http 的路径可以知道对象的版本号，然后将 request body 反序列化成 versioned object;
2. versioned object 转化为 internal object，即没有版本的内部类型，这种资源类型是所有 versioned 类型的超集。只有转化为 internal 后才能适配所有的客户端 versioned object 的校验。
3. Admission Controller 具体的 admit 操作，可以通过这里修改资源对象，例如为 Pod 挂载一个默认的 Service Account 等。
4. API Server internal object validation，校验某个资源对象数据和格式是否合法，例如：Service Name 的字符个数不能超过63等。
5. Admission Controller validate，可以自定义任何的对象校验规则。
6. internal object 转化为 versioned object，并且持久化存储到etcd。
```

#### Default Admission Controller
对pod的改动通过一个被称为Admission Controller的插件来实现。它是apiserver的一部分。 当pod被创建或更新时，它会同步地修改pod。 当该插件处于激活状态(在大多数发行版中都是默认的)，当pod被创建或更新时它会进行以下动作
```
1. 如果该 pod 没有 ServiceAccount 设置，将其 ServiceAccount 设为 default;
2. 保证 pod 所关联的 ServiceAccount 存在，否则拒绝该pod。
3. 如果pod不包含 ImagePullSecrets设置，那么 将 ServiceAccount中的 ImagePullSecrets 信息添加到pod中。
4. 将一个包含用于API访问的token的 volume 添加到pod中。
5. 将挂载于 /var/run/secrets/kubernetes.io/serviceaccount 的 volumeSource 添加到pod下的每个容器中。
```

比如：
```yaml
spec:
  containers:
   - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: default-token-wzm9h
      readOnly: true
      ...
  serviceAccount: default
  serviceAccountName: default
```

#### Admission Controller Plugins Enable/disable
```
--enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,ResourceQuota ...
kube-apiserver --disable-admission-plugins=PodNodeSelector,AlwaysDeny ...
```
#### Default Admission Controller List
```bash
$ kube-apiserver -h | grep enable-admission-plugins
 --enable-admission-plugins strings       admission plugins that should be enabled in addition to default enabled ones (NamespaceLifecycle, LimitRanger, ServiceAccount, Priority, DefaultTolerationSeconds, DefaultStorageClass, PersistentVolumeClaimResize, MutatingAdmissionWebhook, ValidatingAdmissionWebhook, ResourceQuota). Comma-delimited list of admission plugins: AlwaysAdmit, AlwaysDeny, AlwaysPullImages, DefaultStorageClass, DefaultTolerationSeconds, DenyEscalatingExec, DenyExecOnPrivileged, EventRateLimit, ExtendedResourceToleration, ImagePolicyWebhook, Initializers, LimitPodHardAntiAffinityTopology, LimitRanger, MutatingAdmissionWebhook, NamespaceAutoProvision, NamespaceExists, NamespaceLifecycle, NodeRestriction, OwnerReferencesPermissionEnforcement, PersistentVolumeClaimResize, PersistentVolumeLabel, PodNodeSelector, PodPreset, PodSecurityPolicy, PodTolerationRestriction, Priority, ResourceQuota, SecurityContextDeny, ServiceAccount, StorageObjectInUseProtection, ValidatingAdmissionWebhook. The order of plugins in this flag does not matter.
```

#### Admission list
```
AlwaysPullImages：此准入控制器的功能是 修改每个 Pod 的时候都强制重新拉取镜像。
DefaultStorageClass：此准入控制器观察创建PersistentVolumeClaim时不请求任何特定存储类的对象，并自动向其添加默认存储类。这样，用户就不需要关注特殊存储类而获得默认存储类。
DefaultTolerationSeconds：此准入控制器将 Pod 的容忍时间 notready:NoExecute 和 unreachable:NoExecute 默认设置为5分钟。
DenyEscalatingExec：此准入控制器将拒绝在提升了权限的容器中执行 exec 和 attach 命令；比如特权容器。
EventRateLimit (1.13 alpha)：此准入控制器缓解了 API Server 被事件请求流量攻击的问题，限制时间和速率。
ExtendedResourceToleration：此插件有助于创建具有扩展资源的专用节点,为使用这种资源的容器自动调度到专用节点，无需配置容忍策略。
ImagePolicyWebhook：此准入控制器允许后端判断镜像拉取策略，例如配置镜像仓库的密钥。
此准入控制器拒绝任何定义了 requiredDuringSchedulingRequiredDuringExecution 的 AntiAffinity 字段的pod，除了kubernetes.io/hostname 之外的拓扑关键字的。
LimitPodHardAntiAffinityTopology：此准入控制器拒绝任何定义AntiAffinity(requiredDuringSchedulingRequiredDuringExecution)除了kubernetes.io/hostname 之外的拓扑关键字的 pod 。
LimitRanger：此准入控制器将确保所有资源请求不会超过 namespace 的 LimitRange 资源限制。
MutatingAdmissionWebhook （1.9版本中为beta）：该准入控制器调用与请求匹配的任何变更 webhook。匹配的 webhook是串行调用的；如果需要，每个人都可以修改对象。
NamespaceAutoProvision：此准入控制器会检查 namespace 资源上的所有请求，并检查引用的 namespace 是否存在。如果引用的 namespace 不存在就创建该 namespace .
NamespaceExists：与NamespaceAutoProvision 对应， 此许可控制器检查除 Namespace 其自身之外的命名空间资源上的所有请求。如果请求引用的namespace不存在，则拒绝该请求。
NamespaceLifecycle：此准入控制器强制执行正在处于终止状态的 namespace 中不能创建新的对象(pod)，并确保 Namespace 拒绝不存在的请求。此准入控制器还防止缺失三个系统保留的命名空间default、kube-system、kube-public。为了确保执行该过程的完整性，强烈建议运行此准入控制器；
NodeRestriction：该准入控制器限制了 kubelet 可以修改的Node和Pod对象。只允许使用system:nodes组中的凭据的kubelet修改自己的NodeAPI对象，并且只修改 Pod 绑定到其 Node 的API对象。在Kubernetes 1.11+中，不允许 kubelet 更新或删除其NodeAPI对象中的污点。未来版本可能会添加其他限制，以确保kubelet具有正确操作所需的最小权限集。
OwnerReferencesPermissionEnforcement：此准入控制器保护对 metadata.ownerReferences 对象的访问，以便只有对该对象具有“删除”权限的用户才能对其进行更改。此许可控制器还保护对metadata.ownerReferences[x].blockOwnerDeletion 对象的访问，以便只有finalizers 对引用所有者的子资源具有“更新”权限的用户才能更改它。
PodNodeSelector：此准入控制器通过读取命名空间注释和全局配置来限制可在命名空间内使用的节点选择器。
PodPreset：此准入控制器是否允许向匹配的 pod 中注入 PodPreset 中指定的字段.
PodSecurityPolicy：此准入控制器用于创建和修改pod过程中，根据请求的安全上下文和可用的Pod安全策略确定是否允许创建它。
PodTolerationRestriction：此准入控制器首先验证容器的容忍度与其 namespace 级别的node选择器配置侧率之间是否存在冲突，如果存在冲突时拒绝该容器请求。
Priority：此准备入控制器允许控制器使用 priorityClassName 字段并填充优先级的整数值。如果未找到优先级，则拒绝Pod。
ResourceQuota：此准入控制器将检查传入的请求，并确保它不违反命名空间的 ResourceQuota 对象中配置的的任何资源约束，违反了则拒绝请求。
SecurityContextDeny：此准入控制器将拒绝设置某些升级 SecurityContext 字段的pod，比如增加cap，设置特权等。
ServiceAccount：此准入控制器实现 serviceAccounts 的自动化，如果您打算使用Kubernetes ServiceAccount对象，我们强烈建议您使用此准入控制器。
StorageObjectInUseProtection：该插件将kubernetes.io/pvc-protection或kubernetes.io/pv-protection 的 Finalizers 添加到新创建的 PVC 或 PV 。在用户删除 PVC 或 PV 的情况下，PVC或PV不会被移除，直到 PVC 或 PV 保护控制器从PVC或PV中移除Finalizers。有关更多详细信息，请参阅使用中的存储对象保护。比如:
$ kubectl describe pvc test-pvc
Name:          test-pvc
Namespace:     default
StorageClass:  ceph-test
Status:        Bound
...
Finalizers:    [kubernetes.io/pvc-protection]

ValidatingAdmissionWebhook（1.8版本中为alpha；1.9版本中为beta）：该准入控制器调用与请求匹配的任何验证webhook。匹配的webhooks是并行调用的；如果其中任何一个拒绝请求，则请求失败。
```

####  ecommended set of admission controllers
kubernetes 1.10+ 的版本，官方推荐如下:
```bash
--enable-admission-plugins=NamespaceLifecycle,NamespaceExists,LimitRanger,ServiceAccount,DefaultStorageClass,DefaultTolerationSeconds,MutatingAdmissionWebhook,ValidatingAdmissionWebhook,ResourceQuota
```
***PS:***
>--admission-control 配置的控制器列表是有顺序的，越靠前的越先执行，一旦某个控制器返回的结果是reject的，那么整个准入控制阶段立刻结束，所以配置顺序可能会影响你的预期或性能。

#### Dynamic Admission webhook
Kubernetes 提供了很多的准入控制器，但是也不可能满足所有用户的需求,因此Kubernetes提供了Dynamic Admission Controller机制，让大家可以根据个性化需求来开发自己的 Admission Controller,自定义的好处是你一旦更新你的 Admission Controller，不需要重启 kube-apiserver；如果 master 做了HA，更新 kube-apiserver 配置时就不存在幂等问题了，影响很小:
```
Initializers: 允许你可以强制对某些请求或者所有请求都进行校验或者修改操作，比如添加 sidecar、校验 secret 长度或者添加测试 lable 等, 默认disable；
MutatingAdmissionWebhook 允许你在webhook中对object进行mutate修改，默认enable；
ValidatingAdmissionWebhook 只返回 validate 的结果为true 或者 false, 不允许在 webhook 中对 Object 进行修改，默认enable；
```

[更多参考](https://kubernetes.io/blog/2019/03/21/a-guide-to-kubernetes-admission-controllers/)
