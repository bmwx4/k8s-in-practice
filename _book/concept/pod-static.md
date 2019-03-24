# 静态pod

静态Pod是由kubelet进行管理，仅存在于特定Node上的Pod。它们不能通过API Server进行管理，无法与ReplicationController、Deployment或DaemonSet进行关联，并且kubelet也无法对其健康检查。

静态Pod总是由kubelet创建，并且总在kubelet所在的Node上运行。

***创建静态Pod的方式：***  
Kubelet 使用 inotify 机制检测 /etc/kubernetes/manifests 目录（可通过 Kubelet 的 --pod-manifest-path 选项指定）中静态 Pod 的变化，并根据该目录的.yaml或.json文件进行创建操作。在文件发生变化后重新创建相应的 Pod。但有时也会发生修改静态 Pod 的 Manifest 后未自动创建新 Pod 的情景，此时一个简单的修复方法是重启 Kubelet。

```
apiVersion: v1
kind: Pod
metadata:
  name: static-web
  labels:
    name: static-web
spec:
  containers:
  - name: static-web
    image: nginx
    imagePullPolicy: IfNotPresent
    ports:
    - name: web
      containerPort: 80
```

***PS：***  
静态Pod无法通过API Server删除（若删除会变成pending状态），如需删除该Pod则将yaml或json文件从这个目录中删除。
