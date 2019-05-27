# command-line-arguments

#### Docker 与 K8S 中指定可执行程序及参数
通过Dockerfile指定命令和参数:
```
ENTRYPOINT: 定义容器启动时被调用的程序。
CMD: 指定传递给 ENTRYPOINT 的参数.
```
比如下面的 dockerfile:
```
FROM ubuntu:latest

RUN apt-get update ; apt-get -y install fortune
ADD fortuneloop.sh /bin/fortuneloop.sh

ENTRYPOINT ["/bin/fortuneloop.sh"]
CMD ["10"]
```
构建镜像:
```bash
docker build -t luksa/fortune:args .
```
使用 k8s 如何覆盖命令和参数
```yaml
#kugo.yaml
apiVersion: v1
kind: Pod
metadata:
  name: kugo
spec:
  containers:
  - image: bmwx4/kugo:v1.0
    imagePullPolicy: IfNotPresent
    name: kugo
    command: ["/bin/command"]
    args: ["arg1","arg2","arg3"]
```
多数情况下是自定义参数， 命令很少去覆盖，除非针对一些没有定义 ENTRYPOINT 的镜像，比如 busybox;与docker中指定可执行命令和参数方式对比如下:

| Docker | kubernetes | 描述 |
| --- | --- | --- |
|ENTRYPOINT | command | 容器运行的可执行文件 |
|CMD | args | 可执行文件的参数 |

实现改变容器内执行脚本的参数:
```yaml
#fortune2s.yaml
apiVersion: v1
kind: Pod
metadata:
  name: fortune2s
spec:
  containers:
  - image: luksa/fortune:args
    args: ["2"]
    name: html-generator
    volumeMounts:
    - name: html
      mountPath: /var/htdocs
  - image: nginx:alpine
    name: web-server
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html
      readOnly: true
    ports:
    - containerPort: 80
      protocol: TCP
  volumes:
  - name: html
    emptyDir: {}
```
数组也可以表示成:
```yaml
args:
- foo
- bar
- 15
```
