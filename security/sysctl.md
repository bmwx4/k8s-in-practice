# sysctl
通过 sysctl 接口在 Kubernetes 集群中配置和使用内核参数也是很常见的事情，但是在 Kubernetes 把 sysctl 接口定义为 安全的和非安全的;安全 sysctl 参数除了需要设置恰当的命名空间外，在同一 node 上的不同 Pod 之间也必须是 相互隔离的。这意味着在 Pod 上设置安全 sysctl 参数，必须满足以下几点:
```
1. 必须不能影响到node上的其他 Pod
2. 必须不能影响 node 自身的健康
3. 必须不允许使用超出 Pod 的资源限制的 CPU 或内存资源。
```
所有非安全的 sysctl 参数都默认禁用，且必须由集群管理员在每个节点上手动开启。那些设置了不安全 sysctl 参数的 Pod 仍会被调度，但无法正常启动。参考上述警告，集群管理员只有在一些非常特殊的情况下，才可以启用特定的 非安全的 sysctl 参数。如需启用 非安全的 sysctl 参数，请您在每个节点上分别设置 kubelet 命令行参数;

#### 设置 Pod 的 Sysctl 参数
目前，在 Linux 内核中，有许多的 sysctl 参数都是 有命名空间的 。 这就意味着可以为节点上的每个 Pod 分别去设置它们的 sysctl 参数。 在 Kubernetes 中，只有那些有命名空间的 sysctl 参数可以通过 Pod 的 securityContext 对其进行配置。
以下列出有命名空间的 sysctl 参数，在未来的 Linux 内核版本中，此列表可能会发生变化:
```
kernel.shm*,
kernel.msg*,
kernel.sem,
fs.mqueue.*,
net.*
```

**修改kubelet 配置**   
```bash
[Service]
WorkingDirectory=/var/lib/kubelet
ExecStart=/opt/k8s/bin/kubelet \
--allowed-unsafe-sysctls='kernel.msg*,kernel.shmmax,kernel.sem,net.*' \
```
解释一下：
```
kernel.msg*,kernel.shmmax,kernel.sem 允许属于ipc namespace的参数可配；
net.* 允许net namespace的参数配置
```
重启kubelet，配置生效；
```yaml
# sysctl-example.yaml
apiVersion: v1
kind: Pod
metadata:
  name: sysctl-example
spec:
  containers:
  - name: busybox
    image: busybox
    command: [ "sh", "-c", "sleep 1h" ]
    imagePullPolicy: IfNotPresent
  securityContext:
    sysctls:
    - name: kernel.shm_rmid_forced
      value: "0"
    - name: kernel.msgmax
      value: "65536"
```
