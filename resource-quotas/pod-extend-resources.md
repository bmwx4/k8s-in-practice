# 扩展资源
扩展资源作为一种自定义资源，比如将宿主机上的磁盘资源作为一种扩展资源；

首先上报宿主机 node 的磁盘可用资源，如下是 2.7t:
```bash
//为了测试，先启动apiserver的代理
# kubectl proxy  
# cat disk.sh
#!/bin/bash
curl -XPATCH http://127.0.0.1:8001/api/v1/nodes/$1/status -H "Accept: application/json" -H "Content-Type: application/json-patch+json"  -d '
[
{
    "op": "add",
    "path": "/status/capacity/reboot.kubernetes.com~1volume",
    "value": "2700"
}
]'
```

创建第一个pod，使用该扩展资源：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume1
spec:
  containers:
  - name: volume1
    image: nginx
    resources:
      limits:
        cpu: "2"
        reboot.kubernetes.com/volume: 300
        memory: 4Gi
      requests:
        cpu: "0"
        reboot.kubernetes.com/volume: 300
        memory: "0"
```
创建并验证是否可以创建pod；

创建第二个pod，使用该扩展资源：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume2
spec:
  containers:
  - name: volume2
    image: nginx
    resources:
      limits:
        cpu: "2"
        reboot.kubernetes.com/volume: 2700
        memory: 4Gi
      requests:
        cpu: "0"
        reboot.kubernetes.com/volume: 2700
        memory: "0"
```
创建并验证是否可以创建pod；
