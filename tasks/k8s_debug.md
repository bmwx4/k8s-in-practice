# K8S Remote Debug
借住 go程序的debug 方式，以及 goland IDE，我们可以结合这两个工具对k8s组价进行远程debug，对于本地测试开发环境以及 dlv debug工具的使用，我已经整理完了，大家往下看:

### 环境搭建
1. k8s 测试环境搭建可以参考 [K8S本地测试环境搭建](/tasks/local_cluster_up.md)  
2. dlv 环境可以参考 [dlv环境搭建](/tasks/go_debug.md)  
3. goland 安装，官网下载安装个包，破解一下就行了;

### 远程调试
将 k8s 测试环境中的代码，在本地同步一份,切换到你要测试的branch:
```
$ git branch
 master
* v1.15.1
```
### 配置IDE
配置IDE部分主要是配置个远程debug:
![config](/images/config_debug.png)

### 启动k8s
这一步需要单独编译 k8s 组件，而且编译时也要带符号信息的debug版本:
```bash
# make all GOGCFLAGS="-N -l" GOLDFLAGS="" WHAT="cmd/kube-apiserver"
# file _output/bin/kube-apiserver
_output/bin/kube-apiserver: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, not stripped
```
启动 k8s 集群:
```bash
#./hack/local-up-cluster.sh -O
```

启动调试:
```bash
# ps aux | grep kube-apiserver | grep -v grep
root      2510  7.1  5.8 473328 286884 pts/0   Sl+  14:42   1:22 /root/gopath/src/k8s.io/kubernetes/_output/bin/kube-apiserver --authorization-mode=Node,RBAC --runtime-config=settings.k8s.io/v1alpha1 --cloud-provider= --cloud-config= --v=3 --vmodule= --cert-dir=/var/run/kubernetes --client-ca-file=/var/run/kubernetes/client-ca.crt --service-account-key-file=/tmp/kube-serviceaccount.key --service-account-lookup=true --enable-admission-plugins=LimitRanger,ServiceAccount,DefaultStorageClass,DefaultTolerationSeconds,MutatingAdmissionWebhook,ValidatingAdmissionWebhook,ResourceQuota,PodPreset,StorageObjectInUseProtection --disable-admission-plugins= --admission-control-config-file= --bind-address=0.0.0.0 --secure-port=6443 --tls-cert-file=/var/run/kubernetes/serving-kube-apiserver.crt --tls-private-key-file=/var/run/kubernetes/serving-kube-apiserver.key --insecure-bind-address=127.0.0.1 --insecure-port=8080 --storage-backend=etcd3 --etcd-servers=http://127.0.0.1:2379 --service-cluster-ip-range=10.0.0.0/24 --feature-gates=AllAlpha=false --external-hostname=localhost --requestheader-username-headers=X-Remote-User --requestheader-group-headers=X-Remote-Group --requestheader-extra-headers-prefix=X-Remote-Extra- --requestheader-client-ca-file=/var/run/kubernetes/request-header-ca.crt --requestheader-allowed-names=system:auth-proxy --proxy-client-cert-file=/var/run/kubernetes/client-auth-proxy.crt --proxy-client-key-file=/var/run/kubernetes/client-auth-proxy.key --cors-allowed-origins=/127.0.0.1(:[0-9]+)?$,/localhost(:[0-9]+)?$
干掉这个进程:
# kill -15 2510
debug 模式启动:
# dlv --listen=:2345 --headless=true --api-version=2 exec /root/gopath/src/k8s.io/kubernetes/_output/bin/kube-apiserver -- --authorization-mode=Node,RBAC --runtime-config=settings.k8s.io/v1alpha1 --cloud-provider= --cloud-config= --v=3 --vmodule= --cert-dir=/var/run/kubernetes --client-ca-file=/var/run/kubernetes/client-ca.crt --service-account-key-file=/tmp/kube-serviceaccount.key --service-account-lookup=true --enable-admission-plugins=LimitRanger,ServiceAccount,DefaultStorageClass,DefaultTolerationSeconds,MutatingAdmissionWebhook,ValidatingAdmissionWebhook,ResourceQuota,PodPreset,StorageObjectInUseProtection --disable-admission-plugins= --admission-control-config-file= --bind-address=0.0.0.0 --secure-port=6443 --tls-cert-file=/var/run/kubernetes/serving-kube-apiserver.crt --tls-private-key-file=/var/run/kubernetes/serving-kube-apiserver.key --insecure-bind-address=127.0.0.1 --insecure-port=8080 --storage-backend=etcd3 --etcd-servers=http://127.0.0.1:2379 --service-cluster-ip-range=10.0.0.0/24 --feature-gates=AllAlpha=false --external-hostname=localhost --requestheader-username-headers=X-Remote-User --requestheader-group-headers=X-Remote-Group --requestheader-extra-headers-prefix=X-Remote-Extra- --requestheader-client-ca-file=/var/run/kubernetes/request-header-ca.crt --requestheader-allowed-names=system:auth-proxy --proxy-client-cert-file=/var/run/kubernetes/client-auth-proxy.crt --proxy-client-key-file=/var/run/kubernetes/client-auth-proxy.key --cors-allowed-origins='/127.0.0.1(:[0-9]+)?$,/localhost(:[0-9]+)?$'

输出:
API server listening at: [::]:2345
```
这时你回到IDE，设置断点：
![set_bp](/images/set_breakpoint.png)
然后开启debug, 下面会显示已经连接到远程dlv server:
![connect](/images/connect_dlv.png)
剩下的就是你按照框中的几个按钮随心所欲的debug:
```
step into: 单步执行，遇到子函数就进入并且继续单步执行
step over: 在单步执行时，在函数内遇到子函数时不会进入子函数内单步执行，而是将子函数整个执行完再停止，也就是把子函数整个作为一步。
step out: 当单步执行到子函数内时，用step out就可以执行完子函数余下部分，并返回到上一层函数。
Run to cursor: 执行到下一个bp
```
![start](/images/start_debug.png)
