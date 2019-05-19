# StatefulSet Update

StatefulSet 的更新策略（由.spec.updateStrategy.type指定）支持OnDelete 和 RollingUpdate两种：

#### OnDelete
OnDelete 更新策略实现了传统（1.7之前）行为，它也是默认的更新策略。
```bash
kubectl patch statefulset web --type='json' -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/image", "value":"nginx:1.9"}]'
```
当你选择这个更新策略并修改 StatefulSet 的 .spec.template 字段时， StatefulSet 控制器将不会自动的更新Pod。需要手动删除pod；

#### RollingUpdate
RollingUpdate 滚动更新过程也跟 Deployment 大致相同，区别在于：
```
1. 相当于 Deployment 的 maxSurge=0，maxUnavailable=1（其实StatefulSet是不存在这两个配置的）
2. 滚动更新的过程是有序的（逆序），index从N-1到0逐个依次进行，并且下一个Pod创建必须是前一个Pod Ready为前提，下一个Pod删除必须是前一个Pod shutdown并完全删除为前提。
3. 支持部分实例滚动更新，部分不更新，通过.spec.updateStrategy.rollingUpdate.partition来指定一个index分界点。
  3.1 所有ordinal大于等于 partition 指定的值的Pods将会进行滚动更新。
  3.2 所有ordinal小于 partition 指定的值得Pods将保持不变。即使这些Pods被recreate，也会按照原来的pod template创建，并不会更新到最新的版本。
  3.3 特殊地，如果partition的值大于StatefulSet的期望副本数N，那么将不会触发任何Pods的滚动更新。
```
修改 sts 发布更新策略:
```bash
$ kubectl patch statefulset web -p '{"spec":{"updateStrategy":{"type":"RollingUpdate"}}}'
statefulset "web" patched
$ kubectl rollout status sts web
```
rollingUpdate 更新策略会更新一个 StatefulSet 中所有的 Pod，采用与序号索引相反的顺序并遵循 StatefulSet 的保证。

####  partition 实践
设置 partition 为3
```bash
$ kubectl patch statefulset web -p '{"spec":{"updateStrategy":{"type":"RollingUpdate","rollingUpdate":{"partition":3}}}}'
```

发布更新 sts:
```bash
kubectl patch statefulset web -p '{"spec":{"updateStrategy":{"type":"RollingUpdate","rollingUpdate":{"partition":0}}}}'
```

调整 partition 为0
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
