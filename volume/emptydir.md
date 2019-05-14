# emptyDir
如果 Pod 设置了 emptyDir 类型 Volume， Pod 被分配到 Node 上时候，会创建 emptyDir，只要 Pod 运行在 Node 上，emptyDir 都会存在（容器挂掉不会导致 emptyDir 丢失数据），但是如果 Pod 从 Node 上被删除（Pod 被删除，或者 Pod 发生迁移），emptyDir 也会被删除，并且永久丢失。

```yaml
#test-
apiVersion: v1
kind: Pod
metadata:
  name: test-emptydir
spec:
  containers:
  - image: bmwx4/kugo:v1.0
    imagePullPolicy: IfNotPresent
    name: container
    volumeMounts:
    - mountPath: /cache
      name: cache-volume
  volumes:
  - name: cache-volume
    emptyDir: {}
```

### 多容器之间的数据共享

```yaml
# fortune-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: fortune
spec:
  containers:
  - image: luksa/fortune
    imagePullPolicy: IfNotPresent
    name: html-generator
    volumeMounts:
    - name: html
      mountPath: /var/htdocs
  - image: nginx:alpine
    imagePullPolicy: IfNotPresent
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
html-generator 容器负责生产html， 它每十秒启动一次 fortune 命令,
并把内容输出到 /var/htdocs/index.html文件；
```bash
#!/bin/bash
trap "exit" SIGINT
mkdir /var/htdocs

while :
do
  echo $(date) Writing fortune to /var/htdocs/index.html
  /usr/games/fortune > /var/htdocs/index.html
  sleep 10
done
```
web-server 容器负责响应客户端请求；
因为两个容器都挂载了一个相同的volume，所以客户端请求得到的内容，其实是 fortune 命令生成的内容；

验证：
```bash
# kubectl  port-forward fortune 8080:80
# curl localhost:8080
等待几秒之后再次访问....
```
### 指定存储介质
上面的例子是作为卷来使用的 emptyDir, 也是在宿主机的磁盘上创建的一块存储空间，因此性能取决于本地盘的类型。emptyDir 也支持使用内存来充当volume的存储介质，比如：
```yaml
volumes:
- name: html
  emptyDir:
    medium: Memory
```

### gitrepo volume

```yaml
# gitrepo-volume.yaml
apiVersion: v1
kind: Pod
metadata:
  name: gitrepo-volume-pod
spec:
  containers:
  - image: nginx:alpine
    imagePullPolicy: IfNotPresent
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
    gitRepo:
      repository: https://github.com/bmwx4/k8s-in-practice.git
      revision: master
      directory: .
```
gitRepo 和emptyDir 类似，首先将volume初始化一个空目录，然后将git仓库clone到该目录，比如上面的例子，就是 k8s-in-practice 目录；  
验证：
```bash
# kubectl  port-forward gitrepo-volume-pod 8080:80
# socat TCP4-LISTEN:8000,reuseaddr,fork TCP4:127.0.0.1:8080  
通过浏览器访问：
http://192.168.10.232:8000/_book/index.html
```

-----------
fortune 是个很有意思的程序，另外还有其它几个：  
[cmatrix 黑客帝国](https://github.com/abishekvashok/cmatrix)  
[cowsay ASCII字符特殊打印]](https://github.com/piuccio/cowsay)  
[more ...](https://www.binarytides.com/linux-fun-commands/)
