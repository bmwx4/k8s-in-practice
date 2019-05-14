# hostPath
hostPath 是指向worker 节点文件系统上的特定文件或目录， gitRepo和emptyDir的内容都会在pod 被删除时一并被删除， 而hostPath volume的内容不会被删除。如果pod先后都挂载同一个host path的目录的话，后面的pod可以发现前一个pod留下的数据；

```yaml
#test-hostpath.yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-hostpath
spec:
  containers:
  - image: bmwx4/kugo:v1.0
    imagePullPolicy: IfNotPresent
    name: test-container
    volumeMounts:
    - mountPath: /test-pd
      name: test-volume
  volumes:
  - name: test-volume
    hostPath:
      type: Directory
      path: /data
```
那么hostPath都支持哪些类型呢？

| type | action |
| ------ | ------ | ------ |
|  | 空字符串（默认）用于向后兼容，这意味着在挂载 hostPath 卷之前不会执行任何检查|
| DirectoryOrCreate | 如果给定的路径不存在，就按需创建，权限设置为0755 |
| Directory | 如果给定的路径不存在会报错 |
| FileOrCreate | 如果给定的文件路径不存在，就按需创建，权限设置为0644 |
| File | 如果给定的文件路径不存在会报错|
| Socket | 给定的路径下必须存在 UNIX 套接字|
| CharDevice | 给定的路径下必须存在字符设备|
| BlockDevice	 | 给定的路径下必须存在块设备|

***注意:***  
因为hostPath 虽然不会删除数据，但是hostPath和node是强绑定关系，所以，应用hostPath的pod 一旦发生了漂移，就找不到原数据了；
