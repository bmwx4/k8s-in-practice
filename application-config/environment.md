# environment
在pod中设置环境变量与容器的命令设置环境命令相同，环境变量无法在 pod 创建之后被修改，比如通过环境变量配置 fortune 镜像中的时间间隔:
```
FROM ubuntu:latest

RUN apt-get update ; apt-get -y install fortune
ADD fortuneloop.sh /bin/fortuneloop.sh

ENTRYPOINT ["/bin/fortuneloop.sh"]
```
fortuneloop 脚本内容如下:
```bash
#!/bin/bash
trap "exit" SIGINT

echo Configured to generate new fortune every $INTERVAL seconds

mkdir -p /var/htdocs

while :
do
  echo $(date) Writing fortune to /var/htdocs/index.html
  /usr/games/fortune > /var/htdocs/index.html
  sleep $INTERVAL
done
```
下面是在pod中定义环境变量:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: fortune-env
spec:
  containers:
  - image: luksa/fortune:env
    env:
    - name: INTERVAL
      value: "30"
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

验证测试, 可以通过查看容器标准输出日志查看参数传递是否成功:
```bash
$  kubectl logs fortune2s -c html-generator
```
