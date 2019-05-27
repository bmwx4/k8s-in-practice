# downward-api
在生产环境中， 有些业务或者管理员希望通过在容器内部想知道 pod 的IP、主机名或者pod自身的名称(rc/rs等)，此外还有pod的labels和annontations,对于此类问题，k8s 提供了 downward API来解决。 downward API 允许我们通过环境变量或者文件的形式传递 pod 的元数据。比如:

```
pod 的名称
pod 的ip
pod 所在 namespace
pod 运行的node的名称
pod 运行所属的 serviceAccount 名称
每个容器的 CPU、Mem 的使用量
每个容器的 CPU、Mem 的限制量
pod 的 labels
pod 的 annontations
```

#### 通过环境变量暴露元数据
```yaml
# downward-env.yaml
apiVersion: v1
kind: Pod
metadata:
  name: downward
spec:
  containers:
  - name: main
    image: busybox
    imagePullPolicy: IfNotPresent
    command: ["sleep", "1h"]
    resources:
      requests:
        cpu: 15m
        memory: 100Ki
      limits:
        cpu: 100m
        memory: 4Mi
    env:
    - name: POD_NAME
      valueFrom:
        fieldRef:
          fieldPath: metadata.name
    - name: POD_NAMESPACE
      valueFrom:
        fieldRef:
          fieldPath: metadata.namespace
    - name: POD_IP
      valueFrom:
        fieldRef:
          fieldPath: status.podIP
    - name: NODE_NAME
      valueFrom:
        fieldRef:
          fieldPath: spec.nodeName
    - name: SERVICE_ACCOUNT
      valueFrom:
        fieldRef:
          fieldPath: spec.serviceAccountName
    - name: CONTAINER_CPU_REQUEST_MILLICORES
      valueFrom:
        resourceFieldRef:
          resource: requests.cpu
          divisor: 1m
    - name: CONTAINER_MEMORY_LIMIT_KIBIBYTES
      valueFrom:
        resourceFieldRef:
          resource: limits.memory
          divisor: 1Ki
```
环境变量名与环境变量取值对应如下图所示：
![pod-metadata-env](/images/pod-metadata-env.png)

--------------
#### 通过 downwardAPI volume 传递元数据
如果业务或管理员倾向于使用文件的形式读取元数据也可以， 比如定一个 downwardAPI 的volume， 然后挂载容器内就可以了。
```yaml
# downward-volume.yaml
apiVersion: v1
kind: Pod
metadata:
  name: downward
  labels:
    foo: bar
  annotations:
    key1: value1
    key2: |
      multi
      line
      value
spec:
  containers:
  - name: main
    image: bmwx4/kugo:v1.0
    imagePullPolicy: IfNotPresent
    command: ["sleep", "1h"]
    resources:
      requests:
        cpu: 1
        memory: 100Mi
      limits:
        cpu: 2
        memory: 500Mi
    volumeMounts:
    - name: downward
      mountPath: /etc/downward
  volumes:
  - name: downward  #定义一个 downwardAPI volume
    downwardAPI:
      items:
      - path: "podName"
        fieldRef:
          fieldPath: metadata.name
      - path: "podNamespace"
        fieldRef:
          fieldPath: metadata.namespace
      - path: "labels"
        fieldRef:
          fieldPath: metadata.labels
      - path: "annotations"
        fieldRef:
          fieldPath: metadata.annotations
      - path: "containerCpuRequestMilliCores"
        resourceFieldRef:
          containerName: main
          resource: requests.cpu
          divisor: 1m
      - path: "containerMemoryLimitBytes"
        resourceFieldRef:
          containerName: main
          resource: limits.memory
          divisor: 1
```
创建并验证：
```bash
# kubectl create -f downward-volume.yaml
[root@downward /]# ll /etc/downward/   
total 0
lrwxrwxrwx 1 root root 18 May 25 17:37 annotations -> ..data/annotations
lrwxrwxrwx 1 root root 36 May 25 17:37 containerCpuRequestMilliCores -> ..data/containerCpuRequestMilliCores
lrwxrwxrwx 1 root root 32 May 25 17:37 containerMemoryLimitBytes -> ..data/containerMemoryLimitBytes
lrwxrwxrwx 1 root root 13 May 25 17:37 labels -> ..data/labels
lrwxrwxrwx 1 root root 14 May 25 17:37 podName -> ..data/podName
lrwxrwxrwx 1 root root 19 May 25 17:37 podNamespace -> ..data/podNamespace
```
在元数据信息挂载到 /etc/downward 目录；

思考，如何在pod中获取nodename?
```yaml
env:
 - name: MY_NODE_NAME
   valueFrom:
     fieldRef:
       fieldPath: spec.nodeName
```
目前不能通过 volume 方式获取 nodeName；
