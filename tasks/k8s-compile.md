# k8s 组件编译
#### 配置golang开发环境

#### 安装 golang
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
#git clone -b v1.12.4 https://github.com/kubernetes/kubernetes.git
或者：
# git checkout tags/v1.12.8 -b 1.12.8
路径为：
$GOPATH/src/k8s.io/kubernetes
```
#### 依赖gcc
编译之前，需要系统环境里已经存在gcc编译环境:
```bash
# yum install  -y gcc
```

#### 编译各组件
单独编译各个组件:
```bash
make all WHAT=cmd/kube-proxy
make all WHAT=cmd/kubelet
make all WHAT=cmd/kube-apiserver
make all WHAT=cmd/kube-scheduler
make all WHAT=cmd/kube-controller-manager
```
编译成功之后，二进制在当前的 **_output/bin/** 目录下

#### 支持 node 动态调整 CPU 资源 Capacity
**需求:**  
通过node 的 label "cpu-oversold" 来设置放大或缩小 node CPU Capacity 的倍数，要求最大两倍，最小不能超过原始大小；  

**更改代码**    
hack /pkg/kubelet/nodestatus/setters.go 文件，修改上报CPU 时的逻辑:
```go
"import strconv"

func MachineInfo(nodeName string,
	maxPods int,
	podsPerCore int,
	machineInfoFunc func() (*cadvisorapiv1.MachineInfo, error), // typically Kubelet.GetCachedMachineInfo
	capacityFunc func() v1.ResourceList, // typically Kubelet.containerManager.GetCapacity
	devicePluginResourceCapacityFunc func() (v1.ResourceList, v1.ResourceList, []string), // typically Kubelet.containerManager.GetDevicePluginResourceCapacity
	nodeAllocatableReservationFunc func() v1.ResourceList, // typically Kubelet.containerManager.GetNodeAllocatableReservation
	recordEventFunc func(eventType, event, message string), // typically Kubelet.recordEvent
) Setter
  ...
for rName, rCap := range cadvisor.CapacityFromMachineInfo(info) {
				if rName == v1.ResourceCPU {
					if oversold, ok := node.Labels["cpu-oversold"]; ok {
						oversoldFloat, err := strconv.ParseFloat(oversold, 64)
						if err != nil {
							glog.Errorf("Error getting cpu oversold %v", err)
						} else {
							switch {
							case oversoldFloat > float64(2):
								oversoldFloat = float64(2)
							case oversoldFloat < float64(1):
								oversoldFloat = float64(1)
							}
							oversoldResult := int64(oversoldFloat * float64(rCap.Value()))
							if oversoldResult > rCap.Value() {
								rCap.Set(oversoldResult)
							}
						}
					}
				}
      }
      ...
}
```
重新编译kubelet:
```bash
make all WHAT=cmd/kubelet
./_output/bin/kubelet --version
```
这时候你会发现，版本号后边带有 -dirty 后缀，这是因为本地的修改没有提交导致的，因为 make 的时候会读取 commit id， 解决方式只需要把本地的修改做一次本地提交即可:
```bash
git commit -a -m "[feat] xxxxx"
```
然后再次进行编译即可；将新生成的kubelet 替换 老版本kubelet:
```bash
停服:
systemctl stop kubelet
备份:
cp /usr/bin/kublet /usr/bin/kubelet.bac
替换:
cp ./_output/bin/kubelet /usr/bin/kublet
启动新版本:
systemctl start kubelet
```
##### 功能验证
在master上查看node:
给node 打 "cpu-oversold" label， 然后进行验证:
```bash
kubectl  label nodes  192.168.10.242 cpu-oversold=2
```
查看node状态,原本是4 核CPU， 现在是 8 核CPU:
![node-status](/images/node-status.png)

#### it’s ok !!!!
