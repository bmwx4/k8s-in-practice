# Init 容器
Init 容器的基本概念，这是一种专用的容器，在应用程序容器启动之前运行，用来包含一些应用镜像中不存在的实用工具或安装脚本,或者用于等待一些必须完成的初始化动作。

## 理解 Init 容器

[Pod](https://kubernetes.io/docs/concepts/abstractions/pod/) 能够具有多个容器，应用运行在容器里面，但是pod也可以有一个或多个先于应用容器启动的 Init 容器。

Init 容器与普通的容器非常像，除了如下两点：

- Init 容器总是运行到成功完成为止。
- 每个 Init 容器都必须在下一个 Init 容器启动之前成功完成。

如果 Pod 的 Init 容器失败，Kubernetes 会不断地重启该 Pod，直到 Init 容器成功为止。然而，如果 Pod 对应的 `restartPolicy` 为 Never，它不会重新启动。

![init-container](../images/init-container.jpg)

### 如何应用 Init 容器
指定容器为 Init 容器，在 PodSpec 中添加 `initContainers` 字段，以 v1.Container 类型对象的 JSON 数组的形式，还有 app 的 `containers` 数组。 Init 容器的状态在 `status.initContainerStatuses` 字段中以容器状态数组的格式返回（类似 `status.containerStatuses` 字段）。

### 示例

下面列举了 Init 容器的一些用途：
1. 等待一个 Service 创建完成，通过类似如下 shell 命令：
```bash
$  for i in {1..100}; do sleep 1; if ping 192.168.10.232; then exit 0; fi; exit 1
```

2. 将 Pod 注册到白名单服务器，通过在命令中调用 API，类似如下：
```bash
  curl -X POST http://$MANAGEMENT_SERVICE_HOST:$MANAGEMENT_SERVICE_PORT/register -d 'instance=$(<POD_NAME>)&ip=$(<POD_IP>)'
```

3. 在启动应用容器之前等一段时间，使用类似 `sleep 60` 的命令。
4. 克隆 Git 仓库到数据卷。
5. 将配置值放到配置文件中，运行模板工具为主应用容器动态地生成配置文件。例如，在配置文件中存放 POD_IP 值，并使用 Jinja 生成主应用配置文件。

更多详细用法示例，可以在 [StatefulSet 文档](https://kubernetes.io/docs/concepts/abstractions/controllers/statefulsets/) 和 [生产环境 Pod 指南](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/) 中找到。

### 使用 Init 容器
***等待依赖服务***
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  containers:
  - name: myapp-container
    image: busybox
    imagePullPolicy: IfNotPresent
    command: ['sh', '-c', 'echo The app is running! && sleep 3600']
  initContainers:
  - name: init-myservice
    image: busybox
    imagePullPolicy: IfNotPresent
    command: ['sh', '-c', 'until nslookup myservice; do echo waiting for myservice; sleep 2; done;']
  - name: init-mydb
    image: busybox
    command: ['sh', '-c', 'until nslookup mydb; do echo waiting for mydb; sleep 2; done;']
```

```bash
$ kubectl logs myapp-pod -c init-myservice # Inspect the first init container
$ kubectl logs myapp-pod -c init-mydb      # Inspect the second init container
```

下面的 YAML 文件展示了 `mydb` 和 `myservice` 两个 Service：

```yaml
kind: Service
apiVersion: v1
metadata:
  name: myservice
spec:
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
---
kind: Service
apiVersion: v1
metadata:
  name: mydb
spec:
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9377
```

```Bash
$ kubectl create -f services.yaml
service "myservice" created
service "mydb" created
$ kubectl get -f myapp.yaml
NAME        READY     STATUS    RESTARTS   AGE
myapp-pod   1/1       Running   0          9m
```

一旦我们启动了 `mydb` 和 `myservice` 这两个 Service，我们能够看到 Init 容器完成，并且 `myapp-pod` 被创建：
这个例子非常简单，但是应该能够为我们创建自己的 Init 容器提供一些启发。

***初始化配置工作的Pod***
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-demo
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
    ports:
    - containerPort: 80
    volumeMounts:
    - name: workdir
      mountPath: /usr/share/nginx/html
  initContainers:
  - name: install
    image: busybox
    imagePullPolicy: IfNotPresent
    command:
    - wget
    - "-O"
    - "/work-dir/index.html"
    - http://www.baidu.com
    volumeMounts:
    - name: workdir
      mountPath: "/work-dir"
  volumes:
  - name: workdir
    emptyDir: {}
```
***白名单审核***
```
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  containers:
  - name: myapp-container
    image: busybox
    imagePullPolicy: IfNotPresent
    command: ['sh', '-c', 'echo The app is running! && sleep 3600']
  initContainers:
  - name: init-myservice
    image: centos
    imagePullPolicy: IfNotPresent
    command:
    - curl
    - http://www.baidu.com
```
## 总结

Init 容器支持应用容器的全部字段和特性，包括资源限制、数据卷和安全设置。 然而，Init 容器对资源请求和限制的处理稍有不同，在下面 [资源](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/#resources) 处有说明。 而且 Init 容器不支持 Readiness Probe，因为它们必须在 Pod 就绪之前运行完成。

如果为一个 Pod 指定了多个 Init 容器，那些容器会按顺序一次运行一个。只有当前面的 Init 容器必须运行成功后，才可以运行下一个 Init 容器。当所有的 Init 容器运行完成后，Kubernetes 才初始化 Pod 和运行应用容器。


### 思考：资源限制如何做？
