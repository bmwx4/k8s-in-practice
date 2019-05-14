# headless 发现独立的pod
Headless Service也是一种Service，但不同的是会定义spec:clusterIP: None，也就是不需要Cluster IP的Service。

还记得Service的Cluster IP是做什么的吗？对，一个Service可能对应多个EndPoint(Pod)，client访问的是Cluster IP，通过iptables规则转到Real Server，从而达到负载均衡的效果.

那么Headless Service的效果呢？
dns查询会如实的返回2个真实的endpoint.
Headless Service就是没头的Service。有啥用呢？很简单，有时候client想自己来决定使用哪个Real Server，可以通过查询DNS来获取Real Server的信息。

另外，Headless Services还有一个用处。Headless Service的对应的每一个Endpoints，即每一个Pod，都会有对应的DNS域名；这样Pod之间就可以互相访问。我们还是看上面的这个例子。

### 创建headless service
将service 的spec中clusterIP字段设置为None会使服务成为headless服务，因为k8s不会为其分配集群的IP，比如：
```yaml
# kubia-svc-headless.yaml
apiVersion: v1
kind: Service
metadata:
  name: kubia-headless
spec:
  clusterIP: None
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: kubia
```
为了演示清楚， 先使用原svc测试:
```yaml
#kubia-clusterip.yaml
apiVersion: v1
kind: Service
metadata:
  name: kubia
spec:
  clusterIP: 10.254.129.235
  ports:
  - port: 80
    protocol: TCP
    targetPort: 8080
  selector:
    app: kubia
  sessionAffinity: None
  type: ClusterIP
```

```bash
# kubectl  exec limit-pod  -- nslookup kubia
Server:    10.254.0.2
Address 1: 10.254.0.2 kube-dns.kube-system.svc.cluster.local

Name:      kubia
Address 1: 10.254.129.235 kubia.default.svc.cluster.local
```

返回一个 ClusterIP，那么替换成 headless svc再测试一下:
```bash
# kubectl  create -f kubia-svc-headless.yaml
# kubectl  get svc kubia-headless
NAME             TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubia-headless   ClusterIP   None         <none>        80/TCP    24s

```
关注ClusterIP 的值为None， 再来dns测试一下：
```bash
# kubectl  exec limit-pod  -- nslookup kubia-headless
Server:    10.254.0.2
Address 1: 10.254.0.2 kube-dns.kube-system.svc.cluster.local

Name:      kubia-headless
Address 1: 172.30.35.3 172-30-35-3.kubia-headless.default.svc.cluster.local
Address 2: 172.30.14.3 172-30-14-3.kubia-headless.default.svc.cluster.local
```
返回两条记录， 分别为已经准备好的 pod的IP；那么有没有一种机制，可以发现所有的 pod呢？ 即使没有准备就绪的 pod；

在spec下添加如下内容：
```bash
# kubectl  edit svc kubia-headless
spec:
  publishNotReadyAddresses: true
```
再次测试：
```bash
# kubectl  exec limit-pod  -- nslookup kubia-headless                            
Server:    10.254.0.2
Address 1: 10.254.0.2 kube-dns.kube-system.svc.cluster.local

Name:      kubia-headless
Address 1: 172.30.35.2 172-30-35-2.kubia-headless.default.svc.cluster.local
Address 2: 172.30.14.3 172-30-14-3.kubia-headless.default.svc.cluster.local
Address 3: 172.30.35.3 172-30-35-3.kubia-headless.default.svc.cluster.local
```

------------
***PS***

测试DNS时发现如下错误：
```bash
# kubectl exec limit-pod -- nslookup kubernetes.default
Server:         10.254.0.2
Address:        10.254.0.2:53
** server can't find kubernetes.default: NXDOMAIN
*** Can't find kubernetes.default: No answer
```
***原因***  
1. 如果使用busybox镜像测试dns失败，检查配置无误后，再检查一下 busybox 的镜像版本问题，≤ 1.28.4 测试时OK的；
https://github.com/kubernetes/kubernetes/issues/66924
2. 测试dns时，也可以使用 tutum/dnsutils 容器镜像

-------
#### 总结：
- 确保从集群内连接到service 的IP， 而不是从外部；
- 不能通过ping 服务 IP 来判断服务是否可以访问，因为服务的IP是虚拟IP；
- 如果定义了 readiness 探针， 请确保它返回成功；否则pod 不会成为service的一部分，但不会导致pod重建；
- 使用endpoints 来检查并验证相应的后端pod是否OK
- 如果使用域名(FQDN)访问服务不通，先确认使用clusterIP是否可以访问
- 检查是否连接的是服务端口，而不是目标端口
- 检查直连pod的IP访问服务，确保服务自身是OK的
- 通过pod的IP访问不了服务，就使用kubectl log 查看容器日志
