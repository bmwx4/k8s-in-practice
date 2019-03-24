# Pod健康检查

Pod的健康状态由两类探针来检查：LivenessProbe和ReadinessProbe。

### 探针类型
1. livenessProbe (存活探针)
  - 表明容器是否正在运行。
  - 如果存活探测失败，则 kubelet 会杀死容器，并且容器将受到其 [重启策略](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#restart-policy) 的影响。
  - 如果容器不提供存活探针，则默认状态为 Success。
2. readinessProbe (就绪探针)  
  - 指示容器是否准备好服务请求。
  - 如果就绪探测失败，Endpoint 控制器将从与 Pod 匹配的所有 Service 的 Endpoint 中删除该 Pod 的 IP 地址。
  - 初始延迟之前的就绪状态默认为 `Failure`。
  - 如果容器不提供就绪探针，则默认状态为 `Success`。

-----
### Handler

`探针` 是由 [kubelet](https://kubernetes.io/docs/admin/kubelet/) 对容器执行的定期诊断。要执行诊断，kubelet 调用由容器实现的 [Handler](https://godoc.org/k8s.io/kubernetes/pkg/api/v1#Handler)。有三种类型的Handler实现 ：

- ExecAction：在容器内执行指定命令。如果命令退出时返回码为 0 则认为诊断成功。
- TCPSocketAction：对指定端口上的容器的 IP 地址进行 TCP 检查。如果端口打开，则诊断被认为是成功的。
- HTTPGetAction：对指定的端口和路径上的容器的 IP 地址执行 HTTP Get 请求。如果响应的状态码大于等于200 且小于 400，则诊断被认为是成功的。

每次探测都将获得以下三种结果之一：

- 成功：容器通过了诊断。
- 失败：容器未通过诊断。
- 未知：诊断失败，因此不会采取任何行动。

----
### 探针使用方式
- 如果容器中的进程能够在遇到问题或不健康的情况下自行崩溃，则不一定需要存活探针; kubelet 将根据 Pod 的`restartPolicy` 自动执行正确的操作。

- 如果您希望容器在探测失败时被杀死并重新启动，那么请指定一个存活探针，并指定`restartPolicy` 为 Always 或 OnFailure。

- 如果要仅在探测成功时才开始向 Pod 发送流量，请指定就绪探针。在这种情况下，就绪探针可能与存活探针相同，但是 spec 中的就绪探针的存在意味着 Pod 将在没有接收到任何流量的情况下启动，并且只有在探针探测成功后才开始接收流量。

----
### 探针配置
Probe 中有很多精确和详细的配置，通过它们您能准确的控制 liveness 和 readiness 检查：
- initialDelaySeconds：容器启动后第一次执行探测是需要等待多少秒。
- periodSeconds：执行探测的频率。默认是10秒，最小1秒。
- timeoutSeconds：探测超时时间。默认1秒，最小1秒。
- successThreshold：探测失败后，最少连续探测成功多少次才被认定为成功。默认是 1。对于 liveness 必须是 1。最小值是 1。
- failureThreshold：探测成功后，最少连续探测失败多少次才被认定为失败。默认是 3。最小值是 1。  

HTTP probe 中可以给 httpGet设置其他配置项：
- host：连接的主机名，默认连接到 pod 的 IP。您可能想在 http header 中设置 “Host” 而不是使用 IP。
- scheme：连接使用的 schema，默认HTTP。
- path: 访问的HTTP server 的 path。
- httpHeaders：自定义请求的 header。HTTP运行重复的 header。
- port：访问的容器的端口名字或者端口号。端口号必须介于 1 和 65525 之间。

------
### 探针实践

#### Define a liveness command
```yaml
livenessProbe:
  exec:
    command:
    - cat
    - /tmp/healthy
  initialDelaySeconds: 5
  periodSeconds: 5
```
```bash
$ kubectl create -f ../yamls/exec-liveness.yaml
$ kubectl get pod liveness-exec -w
$ kubectl describe pod liveness-exec
```
#### Define a liveness HTTP request
```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
    httpHeaders:
    - name: Custom-Header
      value: Awesome
  initialDelaySeconds: 3
  periodSeconds: 3
```
```bash
$ kubectl create -f ../yamls/http-liveness.yaml
$ kubectl describe pod liveness-http
```

#### Define a TCP liveness & readiness probe
```yaml
readinessProbe:
  tcpSocket:
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10
livenessProbe:
  tcpSocket:
    port: 8080
  initialDelaySeconds: 15
  periodSeconds: 20
```
```bash
kubectl create -f ../yamls/tcp-liveness-readiness.yaml
```

#### Define readiness probes
```yaml
readinessProbe:
  exec:
    command:
    - cat
    - /tmp/healthy
  initialDelaySeconds: 5
  periodSeconds: 5
```

Readiness probe 的 HTTP 和 TCP 的探测器配置跟 liveness probe 一样。唯一的不同是使用 readinessProbe 而不是 livenessProbe。
Readiness 和 livenss probe 可以并行用于同一容器。 使用两者可以确保流量无法到达未准备好的容器，并且容器在失败时重新启动。
