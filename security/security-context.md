# securityContext

在运行容器时，容器启动时加上 --privileged 参数就可以给容器加上特权，使容器和node中root用户一样；在 kubernetes 中可以通过 Security Context 配置，来提高容器的安全能力。kubernetes 提供了三种配置 Security Context的方法：
```
Container-level Security Context：仅应用到指定的容器
Pod-level Security Context：应用到 Pod 内所有容器以及Volume
Pod Security Policies（PSP）：应用到集群内部所有 Pod以及 Volume
```

#### Container-level Security Context
Container-level Security Context 仅应用到指定的容器上。比如设置容器运行在特权模式：
```yaml
# container-level-sec.yaml
apiVersion: v1
kind: Pod
metadata:
  name: container-level-sec
spec:
  containers:
  - name: nginx
    image: nginx:latest
    imagePullPolicy: IfNotPresent
    securityContext:
      privileged: true
```

在生产环境中， 经常会有一些用户提出要开启 特权模式， 但是，开特权可以，一定要划分出独立集群才可以开，或者专机专用的场景，在公共集群里，要坚持不能开,因为开特权需要kube-apiserver和kubelet 同时开启(--allow-privileged=true)才有效，但我发现新版本默认都开启了，最好关掉，然后把需要开启的开启就行,比如：
```bash
kubelet  配置中，关闭 allow-privileged 选项：
[Service]
WorkingDirectory=/var/lib/kubelet
ExecStart=/opt/k8s/bin/kubelet \
  ....
  --allow-privileged=false \
重启kubelet之后再次创建pod:
# kubectl  get pod privileged-c -o wide
NAME           READY   STATUS    RESTARTS   AGE   IP       NODE             NOMINATED NODE
container-level-sec   0/1     Blocked   0          37s   <none>   192.168.10.243   <none>
```
会被 Blocked ， 这是我们希望看到的效果；

---------------

#### Pod-level Security Context
Pod-level Security Context应用到Pod内所有容器，并且还会影响Volume（包括fsGroup和selinuxOptions）。
```yaml
# pod-level-sec.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-level-sec
spec:
  volumes:
  - name: sec-ctx-vol
    emptyDir: {}
  securityContext:
    runAsUser: 1000
    fsGroup: 2000
  containers:
  - name: busybox
    image: busybox
    command: [ "sh", "-c", "sleep 1h" ]
    imagePullPolicy: IfNotPresent
    volumeMounts:
      - name: sec-ctx-vol
        mountPath: /data/demo
```
运行验证:
```
# kubectl  exec -it pod-level-sec sh
/ $ ps
PID   USER     TIME  COMMAND
    1 1000      0:00 sleep 1h
    7 1000      0:00 sh
   15 1000      0:00 ps
------------------------------------
/ $ cd /data/demo
/data/demo $ echo hello > testfile
/data/demo $ ls -l
total 4
-rw-r--r--    1 1000     2000             6 May 25 03:01 testfile
/$ id
```
如果 pod 级别的安全限制和 container 级别的安全限制冲突了，那么 container级别的配置覆盖 pod级别的安全限制，比如：
```yaml
# pod-container-mix-sec.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-container-mix-sec
spec:
  securityContext:
    runAsUser: 1000
  containers:
    - name: busybox
      image: busybox
      command: [ "sh", "-c", "sleep 1h" ]
      imagePullPolicy: IfNotPresent
      securityContext:
        runAsUser: 2000
```

----------  
#### Pod Security Policies（PSP）
Pod Security Policies（PSP）是集群级的Pod安全策略，可以自动为集群内的Pod和Volume设置Security Context。在使用PSP时， 需要API Server开启extensions/v1beta1/podsecuritypolicy，并且配置 PodSecurityPolicyadmission 控制器。  

修改api-server启动项:
```bash
--runtime-config=extensions/v1beta1/podsecuritypolicy=true
--enable-admission-plugins=Initializers,NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota,PodSecurityPolicy
保存&重启 kube-apiserver:
# systemctl  daemon-reload
# systemctl  restart kube-apiserver
重建容器验证
# kubectl  run nginx --image=nginx --image-pull-policy=IfNotPresent
# kubectl  get deployments.
NAME                  DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx                 1         0         0            0           2m20s
查看 kube-controller-manager 报错日志:
# tail -f /var/log/kubernetes/kube-controller-manager.ERROR
E0525 12:21:44.405787  144506 replica_set.go:450] Sync "default/nginx-7ff556d666" failed with pods "nginx-7ff556d666-" is forbidden: unable to validate against any pod security policy: []
E0525 12:21:44.733846  144506 replica_set.go:450] Sync "default/nginx-7ff556d666" failed with pods "nginx-7ff556d666-" is forbidden: unable to validate against any pod security policy: []
E0525 12:21:45.376313  144506 replica_set.go:450] Sync "default/nginx-7ff556d666" failed with pods "nginx-7ff556d666-" is forbidden: unable to validate against any pod security policy: []
E0525 12:21:46.658323  144506 replica_set.go:450] Sync "default/nginx-7ff556d666" failed with pods "nginx-7ff556d666-" is forbidden: unable to validate against any pod security policy: []
E0525 12:21:49.223940  144506 replica_set.go:450] Sync "default/nginx-7ff556d666" failed with pods "nginx-7ff556d666-" is forbidden: unable to validate against any pod security policy: []
E0525 12:21:54.354138  144506 replica_set.go:450] Sync "default/nginx-7ff556d666" failed with pods "nginx-7ff556d666-" is forbidden: unable to validate against any pod security policy: []
```
这时候，需要我们建一个psp，并应用此psp，如下:  
**创建 psp**
```yaml
# PodSecurityPolicy.yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: privileged
spec:
  allowedCapabilities:
  - '*'
  allowPrivilegeEscalation: true
  fsGroup:
    rule: 'RunAsAny'
  hostIPC: true
  hostNetwork: true
  hostPID: true
  hostPorts:
  - min: 0
    max: 65535
  privileged: true
  readOnlyRootFilesystem: false
  runAsUser:
    rule: 'RunAsAny'
  seLinux:
    rule: 'RunAsAny'
  supplementalGroups:
    rule: 'RunAsAny'
  volumes:
  - '*'
```

**然后需要创建一个ClusteRole，并且赋予该Role对上述psp对使用权**
```yaml
#privileged-psp.yaml
# Cluster role which grants access to the privileged pod security policy
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: privileged-psp
rules:
- apiGroups:
  - policy
  resourceNames:
  - privileged
  resources:
  - podsecuritypolicies
  verbs:
  - use
```

**再建立一个 rolebinding 到 admin用户:**
```yaml
#rolebing-privileged-psp.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: rolebing-privileged-psp
  namespace: default
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: privileged-psp
subjects:
- kind: Group
  apiGroup: rbac.authorization.k8s.io
  name: system:authenticated
```

验证：
```bash
#  kubectl  run nginx --image=nginx --image-pull-policy=IfNotPresent
# kubectl  get deployments.
NAME                  DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx                 1         1         1            1           3s
# kubectl  get psp
NAME         PRIV   CAPS   SELINUX    RUNASUSER   FSGROUP    SUPGROUP   READONLYROOTFS   VOLUMES
privileged   true   *      RunAsAny   RunAsAny    RunAsAny   RunAsAny   false            *
```
----------

#### Set capabilities for a Container
和我们接触容器时的 capabilities 是一样的， 也可以通过 [Linux capabilities](http://man7.org/linux/man-pages/man7/capabilities.7.html) 了解更多，这里，举一个例子，比如用户想添加iptables策略，那么需要平台给用户提供 NET_ADMIN 的能力，操作之前，可以先了解下当前容器具备哪些 capabilities:
```bash
kubectl  exec -it pod-level-sec sh
cd /proc/1
cat status|grep Cap
...
CapPrm: 0000000000000000
CapEff: 0000000000000000
....
```
然后再加一个 NET_ADMIN：
```yaml
# capabilities.yaml
apiVersion: v1
kind: Pod
metadata:
  name: capabilities
spec:
  containers:
  - name: busybox
    image: busybox
    command: [ "sh", "-c", "sleep 1h" ]
    imagePullPolicy: IfNotPresent
    securityContext:
      capabilities:
        add: ["NET_ADMIN"]
```
再次查看 capabilities:
```bash
kubectl  exec -it capabilities sh
cd /proc/1
cat status|grep Cap
...
CapPrm: 00000000a80435fb
CapEff: 00000000a80435fb
....
```
也可以在容器内部通过 iptables 命令来验证测试；  
```bash
kubectl  exec -it capabilities -- iptables -L
```

capabilities也包含了3个集合：
```
这个集合定义了线程所能够拥有的特权的上限。换句话说，如果某个capability不在Permitted集合中，那么该线程便不能进行这个capability所对应的特权操作。Permitted集合是Inheritable和Effective集合的的超集。
Inheritable:
当执行exec()系运行其他命令时，能够被新命令继承的capabilities，被包含在Inheritable集合中。
Effective:
这仅仅是一个bit。如果设置开启，那么在运行exec后，Permitted集合中新增的capabilities会自动出现在Effective集合中；否则不会出现在Effective集合中。对于一些旧的可执行文件，由于其不会调用capabilities相关函数设置自身的Effective集合，所以可以将该可执行文件的Effective bit开启，从而将Permitted集合中的capabilities自动添加到Effective集合中。内核检查该线程是否可以进行特权操作时，检查的对象便是Effective集合。Permitted集合定义了上限。线程可以删除Effective集合中的某capability，随后在需要时，再从Permitted集合中恢复该capability，以此达到临时禁用capability的功能。
```
