# Pod 的生命周期

## Pod phase

Pod 的 `status` 字段是一个 PodStatus 对象，PodStatus中有一个 `phase` 字段。

Pod 的相位（phase）是 Pod 在其生命周期中的简单宏观概述。该阶段并不是对容器或 Pod 的综合汇总，也不是为了做为综合状态机。

Pod 相位的数量和含义是严格指定的。除了本文档中列举的状态外，不应该再假定 Pod 有其他的 `phase` 值。

下面是 `phase` 可能的值：

- 挂起（Pending）：Pod 已被 Kubernetes 系统接受，但有一个或者多个容器镜像尚未创建。等待时间包括调度 Pod 的时间和通过网络下载镜像的时间，这可能需要花点时间。
- 运行中（Running）：该 Pod 已经绑定到了一个节点上，Pod 中所有的容器都已被创建。至少有一个容器正在运行，或者正处于启动或重启状态。
- 成功（Succeeded）：Pod 中的所有容器都被成功终止，并且不会再重启。
- 失败（Failed）：Pod 中的所有容器都已终止了，并且至少有一个容器是因为失败终止。也就是说，容器以非0状态退出或者被系统终止。
- 未知（Unknown）：因为某些原因无法取得 Pod 的状态，通常是因为与 Pod 所在主机通信失败。

下图是Pod的生命周期示意图，从图中可以看到Pod状态的变化。

![Pod的生命周期示意图（图片来自网络）](../images/kubernetes-pod-life-cycle.jpg)

----
## Pod 状态

Pod 有一个 PodStatus 对象，其中包含一个 PodCondition 数组。 PodCondition 数组的每个元素都有一个 `type` 字段和一个 `status` 字段。`type` 字段是字符串，可能的值有 PodScheduled、Ready、Initialized、Unschedulable和ContainersReady。`status` 字段是一个字符串，可能的值有 True、False 和 Unknown。

```
lastProbeTime：Pod condition最后一次被探测到的时间戳。
lastTransitionTime：Pod最后一次状态转变的时间戳。
message：状态转化的信息，一般为报错信息，例如：containers with unready status: [c-1]。
reason：最后一次状态形成的原因，一般为报错原因，例如：ContainersNotReady。
status：包含的值有 True、False 和 Unknown。
type：Pod状态的几种类型。
```

其中type字段包含以下几个值：  
```
PodScheduled：Pod已经被调度到运行节点。
Ready：Pod已经可以接收请求提供服务。
Initialized：所有的init container已经成功启动。
Unschedulable：无法调度该Pod，例如节点资源不够。
ContainersReady：Pod中的所有容器已准备就绪。
```
***PS：***  
有关 Pod 和 容器状态的详细信息，请参阅 PodStatus 和 ContainerStatus。 Pod 状态信息取决于当前的 ContainerState。

----
## 重启策略
PodSpec 中有一个 `restartPolicy` 字段，Pod 通过这个字段设定重启策略， 策略类型为 Always、OnFailure 和 Never。默认为 Always。
- Always： 当容器失效时，由kubelet自动重启该容器
- OnFailure: 当容器终止运行且退出码不为0时，由kubelet自动重启该容器
- Never: 不论容器运行状态如何，kubelet都不会重启该容器

说明：  
可以管理Pod的控制器有 Deployment，Statefulset, Job，DaemonSet，及kubelet（静态Pod）。

Deployment，Statefulset和DaemonSet：必须设置为Always，需要保证该容器持续运行。
Job：OnFailure或Never，确保容器执行完后不再重启。
kubelet：在Pod失效的时候重启它，不论RestartPolicy设置为什么值，并且不会对Pod进行健康检查。

 `restartPolicy` 适用于 Pod 中的所有容器。`restartPolicy` 仅指通过同一节点上的 kubelet 重新启动容器。失败的容器由 kubelet 以五分钟为上限的指数退避延迟（10秒，20秒，40秒...）重新启动，并在成功执行十分钟后重置。如 [Pod 文档](https://kubernetes.io/docs/user-guide/pods/#durability-of-pods-or-lack-thereof) 中所述，一旦绑定到一个节点，Pod 将永远不会重新绑定到另一个节点。

----
## Pod 的生命
Pod的生命周期一般通过Controler 的方式管理，每种Controller都会包含PodTemplate来指明Pod的相关属性，Controller可以自动对pod的异常状态进行重新调度和恢复，除非通过Controller的方式删除其管理的Pod，不然kubernetes始终运行用户预期状态的Pod。

***控制器的分类：***  
- 使用 Job运行预期会终止的 Pod，例如批量计算。Job 仅适用于重启策略为 OnFailure 或 Never 的 Pod。
- 对预期不会终止的 Pod 使用 Deployment，Statefulset，例如 Web 服务器。
- 提供特定于机器的系统服务，使用 DaemonSet 为每台机器运行一个 Pod 。

如果节点死亡或与集群的其余部分断开连接，则 Kubernetes 将应用一个策略将丢失节点上的所有 Pod 的 `phase` 设置为 `Failed`。

----
### Pod状态转换
**常见的状态转换**

| Pod的容器数 | Pod当前状态 | 发生的事件    | Pod结果状态              |                         |                     |
| ------- | ------- | -------- | -------------------- | ----------------------- | ------------------- |
|         |         |          | RestartPolicy=Always | RestartPolicy=OnFailure | RestartPolicy=Never |
| 包含一个容器  | Running | 容器成功退出   | Running              | Succeeded               | Succeeded           |
| 包含一个容器  | Running | 容器失败退出   | Running              | Running                 | Failure             |
| 包含两个容器  | Running | 1个容器失败退出 | Running              | Running                 | Running             |
| 包含两个容器  | Running | 容器被OOM杀掉 | Running              | Running                 | Failure             |

#### 容器运行时内存超出限制：
  - 容器以失败状态终止。
  - 记录 OOM 事件。
  - 如果 `restartPolicy` 为：
    - Always：重启容器；Pod `phase` 仍为 Running。
    - OnFailure：重启容器；Pod `phase` 仍为 Running。
    - Never: 记录失败事件；Pod `phase` 仍为 Failed。

#### Pod 正在运行，磁盘故障：
  - 杀掉所有容器。
  - 记录适当事件。
  - Pod `phase` 变成 Failed。
  - 如果使用控制器来运行，Pod 将在别处重建。

#### Pod 正在运行，其节点挂掉。
  - 节点控制器等待直到超时。
  - 节点控制器将 Pod `phase` 设置为 Failed。
  - 如果是用控制器来运行，Pod 将在别处重建。

参考：https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/
