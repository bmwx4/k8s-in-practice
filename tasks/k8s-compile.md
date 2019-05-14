# k8s 组件编译
#### 配置golang开发环境

下载golang并解压缩：
```bash
wget https://dl.google.com/go/go1.11.10.linux-amd64.tar.gz
tar zxvf go1.11.10.linux-amd64.tar.gz
```

添加环境变量：
```bash
export GOROOT="/root/go"
export GOPATH="/root/gopath"
export PATH=$GOROOT/bin:$GOPATH/bin:$PATH
```

克隆 k8s到本地:
```bash
git clone -b v1.12.4 https://github.com/kubernetes/kubernetes.git
路径为：
$GOPATH/src/k8s.io/kubernetes
```
单独编译各个组件:
```bash
make all WHAT=cmd/kube-proxy
make all WHAT=cmd/kubelet
make all WHAT=cmd/kube-apiserver
make all WHAT=cmd/kube-scheduler
make all WHAT=cmd/kube-controller-manager
```
编译成功之后，二进制在当前的 **_output/bin/** 目录下
