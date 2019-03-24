# Pod HOOK

## Pod HOOK 给容器生命周期设置操作事件

Kubernetes支持预启动和预结束事件。 Kubernetes在容器启动的时候发送预启动事件，在容器结束的时候发送预结束事件。

### 定义预启动和预结束事件操作
下面是Pod的配置文件：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: lifecycle-demo
spec:
  containers:
  - name: lifecycle-demo-container
    image: nginx
    imagePullPolicy: IfNotPresent
    lifecycle:
      postStart:
        exec:
          command: ["/bin/sh", "-c", "echo Hello from the postStart handler > /usr/share/message"]
      preStop:
        exec:
          command: ["/usr/sbin/nginx","-s","quit"]
```

在这个配置文件里，你可以看到postStart命令在容器目录/usr/share下写了一个message文件， preStop命令停止容器。这在容器被因错误而结束时很有帮助。

验证 Pod:
```bash
$ kubectl create -f ../yamls/lifecycle-events.yaml
$ kubectl get pod lifecycle-demo
$ kubectl exec -it lifecycle-demo -- /bin/bash
在shell里，验证postStart是否创建了message文件：
root@lifecycle-demo:/# cat /usr/share/message
```

postStart/postSop 支持两种类型的HOOK：
1. exec：执行一段命令
2. HTTP：发送HTTP请求。


### 总结
Kubernetes在容器创建之后就会马上发送postStart事件，但是并没法保证一定会 这么做，它会在容器入口被调用之前调用postStart操作，因为postStart的操作跟容器的操作是异步的，而且Kubernetes控制台会锁住容器直至postStart完成，因此容器只有在 postStart 操作完成之后才会被设置成为RUNNING状态。

Kubernetes在容器结束之前发送preStop事件，并会在preStop操作完成之前一直锁住容器状态，除非Pod优雅退出时间过期了。


### 思考
postStart 是否可以应用于给业务容器添加Mysql，Redis白名单机制？
