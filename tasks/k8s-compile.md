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

#### 支持 node 动态调整 CPU 资源 Capacity
hack /pkg/kubelet/nodestatus/setters.go 文件，修改上报CPU 时的逻辑:
```go
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
