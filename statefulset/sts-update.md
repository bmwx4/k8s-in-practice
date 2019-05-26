# StatefulSet Update

StatefulSet 的更新策略（由.spec.updateStrategy.type指定）, 该字段支持OnDelete 和 RollingUpdate两种；

#### OnDelete
OnDelete 可防止控制器自动更新其 Pod。您必须手动删除 Pod，以使控制器创建新 Pod 来反映您的更改。如果您希望手动更新 Pod，则 OnDelete 非常有用。 它也是默认的更新策略。
通过如下命令可以在线修改 sts 的.spec.template 字段 :
```bash
kubectl patch statefulset web --type='json' -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/image", "value":"nginx:1.9"}]'
```
这时你会发现修改了 StatefulSet 的 .spec.template 字段之后， StatefulSet 控制器将不会自动的更新Pod。需要手动删除 pod ，重建pod之后才应用最新的
template ，这是符合预期的；如果想让其自动滚动更新，请尝试下面的 RollingUpdate 更新策略；

#### RollingUpdate
RollingUpdate 实现了 StatefulSet 中的 Pod 的自动滚动更新。RollingUpdate 策略使控制器删除并重新创建每个 Pod，并且一次只能创建一个 Pod。直到更新的 Pod 运行并就绪，这个 Pod 才可以替代上一个 Pod。StatefulSet 的滚动更新过程也跟 Deployment 大致相同，区别在于：
```
1. 相当于 Deployment 的 maxSurge=0，maxUnavailable=1（其实StatefulSet是不存在这两个配置的）
2. 滚动更新的过程是有序的（逆序），index从N-1到0逐个依次进行，并且下一个Pod创建必须是前一个Pod Ready为前提，下一个Pod删除必须是前一个Pod shutdown并完全删除为前提。
3. 支持部分实例滚动更新，部分不更新，通过.spec.updateStrategy.rollingUpdate.partition来指定一个index分界点。
  3.1 所有ordinal大于等于 partition 指定的值的Pods将会进行滚动更新。
  3.2 所有ordinal小于 partition 指定的值得Pods将保持不变。即使这些Pods被recreate，也会按照原来的pod template创建，并不会更新到最新的版本。
  3.3 特殊地，如果partition的值大于StatefulSet的期望replicas数N，那么将不会触发任何Pods的滚动更新。
```
** 在线修改 sts 发布更新策略:**
```bash
$ kubectl patch statefulset web -p '{"spec":{"updateStrategy":{"type":"RollingUpdate"}}}'
statefulset "web" patched
$ kubectl rollout status sts web
```
然后， 更改 StatefulSet 的 spec.template， 例如，可以使用 kubectl set 命令来修改容器的镜像，比如:
```bash
kubectl set image statefulset web nginx=nginx:latest
```
可以在另一个终端持续观察 pod 变化:
```bash
$ kubectl get pod -o wide -w
```
RollingUpdate 更新策略会更新一个 StatefulSet 中所有的 Pod，采用与序号索引相反的顺序并遵循 StatefulSet 的保证。
>注意：虽然控制器在一个 Pod 处于“正在运行且就绪”状态之前不会继续更新下一个 Pod，但它会将更新失败的所有 Pod 恢复到其当前版本。已收到更新的 Pod 将恢复为更新版本，尚未收到更新的 Pod 依然为先前版本。

####  对 RollingUpdate 实行分区
可以指定 StatefulSet 的 RollingUpdate 字段的 partition 参数来实行滚动更新分区。如果要进行更新、发布 canary 版或进行阶段发布，都可以利用分区。  
例如，要对 web StatefulSet 进行分区，请运行以下命令：
```bash
$ kubectl patch statefulset web -p '{"spec":{"updateStrategy":{"type":"RollingUpdate","rollingUpdate":{"partition":3}}}}'
```
这会导致序pod序号大于等于 3 的 Pod 进行更新。观察容器都ready之后，继续发布更新 sts:
```bash
kubectl patch statefulset web -p '{"spec":{"updateStrategy":{"type":"RollingUpdate","rollingUpdate":{"partition":1}}}}'
```
最后调整 partition 为0
```bash
$ kubectl patch statefulset web -p '{"spec":{"updateStrategy":{"type":"RollingUpdate","rollingUpdate":{"partition":0}}}}'
```

#### StatefulSet 的Pod 管理策略
Kubernetes 1.7+，StatefulSet开始支持Pod Management Policy配置，提供以下两种配置：
```
1. OrderedReady，StatefulSet的Pod默认管理策略，就是逐个的、顺序的进行部署、伸缩，也是默认的策略。
2. Parallel，支持并行创建或者删除同一个StatefulSet下面的所有Pods，并不会逐个的、顺序的等待前一个操作确保成功后才进行下一个Pod的处理。其实用这种管理策略的场景非常少。
```
**在线修改 web 集群的Pod Management Policy**  
```bash
$ kubectl get sts web -o yaml >web.yaml
更改 podManagementPolicy: Parallel
$ kubectl delete sts web --cascade=false
$ kubectl create -f web.yaml
```
