## 本地测试环境搭建
我们经常想在本地用一个虚拟机来方便的进行k8s测试，或者你也可以远程调试，调试方式请[参考](/tasks/k8s_debug.md)，

### 环境准备
对于一个k8s的运行环境，在k8s-install 那部分的第一章节已经介绍到了，按照[系统初始化和全局变量
](https://github.com/bmwx4/k8s-install/blob/v1.12/os-init.md)来准备就可以，另外需要安装docker, etcd, 证书生成工具以及 golang 的开发环境.
### 1.  docker安装
可以参考这篇文章: https://github.com/bmwx4/k8s-install/blob/v1.12/docker.md ， 也可以使用yum 安装并启动；

### 2. etcd 安装
不需要你去把etcd服务run起来，把二进制放入到你的$PATH里就可以了:
```bash
$ wget https://github.com/coreos/etcd/releases/download/v3.3.11/etcd-v3.3.11-linux-amd64.tar.gz
$ tar -C etcd-v3.3.11-linux-amd64.tar.gz /usr/local/
$ cp /usr/local/etcd-v3.3.11-linux-amd64/etcd* /usr/bin
```
### 3. golang 开发环境安装
```bash
$ cd ~
$ wget https://dl.google.com/go/go1.12.7.linux-amd64.tar.gz
$ tar xvf go1.12.7.linux-amd64.tar.gz
$ cat ~/.bash_profile
  export GOPATH=$HOME/gopath
  export GOROOT=$HOME/go
  export PATH=$GOROOT/bin:$GOPATH/bin:$PATH
$ mkdir gopath/pkg gopath/bin gopath/src
```
### 4. openssl 安装
检查是否已经安装了openssl，如果没有安装，yum 安装即可;
```bash
# yum info openssl
已加载插件：fastestmirror, langpacks
Loading mirror speeds from cached hostfile
 * base: mirrors.aliyun.com
 * extras: mirrors.aliyun.com
 * updates: mirrors.aliyun.com
已安装的软件包
名称    ：openssl
架构    ：x86_64
时期       ：1
版本    ：1.0.2k
发布    ：12.el7
大小    ：814 k
源    ：installed
```
### 5. CFSSL 安装
cfssl 包含两个工具，证书生成和解析: cfssl, cfssljson； 保证在我们的环境变量中能找到即可:
```bash
$ go get -u github.com/cloudflare/cfssl/cmd/...
```
### 6. k8s 代码clone
需要一份源代码， 因为local up 启动环境是需要进行编译这份代码，生成各个k8s组件的， 那么，你就可以按你所想，切换不同的代码分支，进行不同版本的debug或者test了,比如我想测试的版本是 1.15.1；
```bash
git clone https://github.com/kubernetes/kubernetes.git $GOPATH/src/k8s.io/kubernetes
cd $GOPATH/src/k8s.io/kubernetes
$ git checkout  -b v1.15.1  v1.15.1
```

### 7. 启动集群
直接执行 hack/local-up-cluster.sh 脚本即可启动一个轻量的k8s集群，但由于 local-up-cluster.sh 默认使用的是 hyperkube 镜像在容器中启动 k8s 服务，这样不方便后续我们使用dlv调试代码，这里修改 [local-up-cluster.sh](https://github.com/bmwx4/local-up-cluster/blob/master/local-up-cluster.sh) 脚本，直接编译出带调试信息的可执行文件，并在主机中启动k8s相关进程。
启动单机集群: 第一次执行下面的命令编译并启动集群
```bash
./hack/local-up-cluster.sh
.... 省略中间过程
export KUBERNETES_PROVIDER=local

cluster/kubectl.sh config set-cluster local --server=https://localhost:6443 --certificate-authority=/var/run/kubernetes/server-ca.crt
cluster/kubectl.sh config set-credentials myself --client-key=/var/run/kubernetes/client-admin.key --client-certificate=/var/run/kubernetes/client-admin.crt
cluster/kubectl.sh config set-context local --cluster=local --user=myself
cluster/kubectl.sh config use-context local
cluster/kubectl.sh
```
### 8. 验证
```bash
设置自动补齐:
#source <(kubectl completion bash)
获取组件状态:
# kubectl get componentstatuses
NAME                 STATUS    MESSAGE             ERROR
etcd-0               Healthy   {"health":"true"}
scheduler            Healthy   ok
controller-manager   Healthy   ok
创建个pod:
[root@localhost ~]# kubectl  get pod
NAME                     READY   STATUS    RESTARTS   AGE
nginx-7bb7cd8db5-ztqtl   1/1     Running   0          3m31s
[root@localhost ~]# kubectl  get deployments.
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
nginx   1/1     1            1           3m37s
创建个服务:
# kubectl  expose deployment nginx --type=NodePort --port=80
测试服务: 我的vm IP 是 192.168.10.233
# curl 192.168.10.233:31847
```

### PS
因为第一次启动需要编译各个组件，后续，如果启动时不想经历各个组件时，可是使用 '-O' 选项， 是大写字母O，不是数字 0 哦；比如:
```bash
# ./hack/local-up-cluster.sh -O
```

到此本地开发测试环境就OK了

### 参考
1. https://github.com/kubernetes/community/blob/master/contributors/devel/running-locally.md
