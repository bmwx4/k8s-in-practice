# RBAC
RBAC  是基于角色的访问控制（Role-Based Access Control）,在 RBAC 模型中，权限与角色相关联，用户通过成为适当角色的成员而得到这些角色的权限。这就极大地简化了权限的管理。这样管理都是层级相互依赖的，权限赋予给角色，而把角色又赋予用户，这样的权限设计清晰，管理起来也很方便。

## RBAC 在k8s中的应用
在日常运维过程中，有些用户或者平台想访问 Kubernetes 中的一些对象(全部是不可能的)，那么，我们是肯定不会把admin这类超级账户给放出去的，那怎么办呢？k8s给我们提供了授权访问模式，比如 有ABAC（基于属性的访问控制）、RBAC（基于角色的访问控制）、Webhook、Node、AlwaysDeny（一直拒绝）和AlwaysAllow（一直允许）这6种模式。从1.8开始，RBAC已作为稳定的功能。通过设置 –authorization-mode=RBAC，启用RABC。在RABC API中，通过如下的步骤进行授权：
```
1）定义角色(Role and ClusterRole)：在定义角色时会指定此角色对于资源访问控制的一系列规则；
2) 定义主体("User", "Group" or "ServiceAccount"): 客户端标识；
3）绑定角色RoleBinding and ClusterRoleBinding：将主体与角色进行绑定，对用户进行访问授权。
```
![rbac](/images/rbac.png)

### Role vs ClusterRole

#### Role：
ole是一系列的权限的集合，例如一个Role可以包含读取 Pod 的权限和列出 Pod 的权限,Role只能授予单个 namespace 中资源的访问权限,比如：
```yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""] # "" indicates the core API group
  resources: ["pods"]
  #resourceNames: ["my-pod"]
  verbs: ["get", "watch", "list"]
```

#### ClusterRole
ClusterRole授权 >= Role授予（与Role类似），但ClusterRole属于集群级别对象,比如:
```
集群范围（cluster-scoped）的资源访问控制（如：节点访问权限）
非资源类型（如"/healthz"）
所有namespaces 中的namespaced 资源（如 pod）
```
** 所有namespaces 中的 namespaced 资源（如 pod）**
```yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: cluster-pod-reader
rules:
- apiGroups: [""] # "" indicates the core API group
  resources: ["pods"]
  #resourceNames: ["my-pod"]
  verbs: ["get", "watch", "list"]
```
** 允许读取核心 Group 中资源 “Node”（Node属于集群范围，则必须将 ClusterRole 绑定 ClusterRoleBinding ): **
```yaml
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list", "watch"]
```

**允许非资源型 "/healthz" 和所有子路径进行 “GET”和“POST”请求,必须将ClusterRole绑定ClusterRoleBinding:**

```yaml
rules:
- nonResourceURLs: ["/healthz", "/healthz/*"] # '*' in a nonResourceURL is a suffix glob match
  verbs: ["get", "post"]
```

其中，关于 RBAC 的权限定义部分主要有三个字段：
```
1. apiGroups: 指定哪个 API 组下的权限,比如：apps，可以使用该命令获取apiGroups，如果为空字符串，代表就是core下的api: kubectl api-resources -o wide
2. resources: 该apiGroups组下具体资源，比如 pod,service,secret 等
3. verbs: 指对该资源具体执行哪些动作,如 get, delete,list 等
```

### RoleBinding vs ClusterRoleBinding
#### RoleBinding
RoleBinding是将Role中定义的权限授予给用户或用户组。RoleBinding 适用于某个命名空间内授权，比如以下 RoleBinding 是将 default 命名空间的 pod-reader Role 授予 bmw 用户，此后 bmw 用户在 default 命名空间中将具有 pod-reader 的权限
```yaml
# This role binding allows "bmw" to read pods in the "default" namespace.
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: User  # May be "User", "Group" or "ServiceAccount"
  name: bmw
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```
***PS：*** ** RoleBinding还可以roleRef到 ClusterRole **  
相当于 kubectl 命令:
```bash
$ kubectl create rolebinding read-pods --role=pod-reader --user=bmw --namespace=default
$ kubectl  get rolebindings.rbac.authorization.k8s.io  -o wide
其它类似 subjects:
$ kubectl create rolebinding read-pods-svc --role=pod-reader --serviceaccount=default:bmw-svc --namespace=default
$ kubectl create rolebinding read-pods-group --role=pod-reader --group=bmw-group --namespace=default
```
基本解释: 将 bmw 用户(serviceAccount,group)在 default 命名空间中将具有 pod-reader 的权限

#### ClusterRoleBinding
ClusterRoleBinding 适用于集群范围内的授权，也就是说可以跨namespace。比如以下 ClusterRoleBinding 是将 bmw 用户在任意namespace中都具有 pod-reader 的权限：
```yaml
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: read-pods-global
subjects:
- kind: User
  name: bmw
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-pod-reader
  apiGroup: rbac.authorization.k8s.io
```
相当于 kubectl 命令:
```bash
$ kubectl create clusterrolebinding read-pods-global --clusterrole=cluster-pod-reader --user=bmw
$ kubectl  get clusterrolebindings.rbac.authorization.k8s.io  -o wide
```

### Subject(主体)
RBAC授权中的主体可以是组，用户或者服务帐户。用户通过字符串表示，比如 "bmw"、"bmw@linux.org" 等;具体的形式取决于管理员在认证模块中所配置的用户名。system:被保留作为用来Kubernetes系统使用，因此不能作为用户的前缀。组也有认证模块提供，格式与用户类似。
#### 在角色绑定主体的例子：
```yaml
subjects:
- kind:User
  name:"bmw@linux.org"
  apiGroup:rbac.authorization.k8s.io
```

#### 名称为“bmw-group”的组：
```yaml
subjects:
- kind:Group
  name:"bmw-group"
  apiGroup:rbac.authorization.k8s.io
```

#### 名称为“bmw-sv”的ServiceAccount：
```yaml
subjects:
- kind:ServiceAccount
  name:"bmw-svc"
  apiGroup:rbac.authorization.k8s.io
```

### 实践
假如我们想给用户 bmw 开通一个只可以查看集群中default namespace中的 pods,services 权限，该如何实现呢？一般分如下几个步骤：
```
1. 定义个角色(Role/ClusterRole)，角色包含资源(resources)和动作(verbs)
2. 分别以 User,Group, ServiceAccount 模式定义一个主体
3. 定义一个 (RoleBinding/ClusterRoleBinding), 把 subjects 与 Role/ClusterRole 进行关联
4. 配置 kubeconfig,使用 kubectl 验证
```

** 1. 创建角色**
```yaml
# bmw-svc-role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
rules:
- apiGroups:
  - ""
  resources:
  - pods
  - pods/status
  - pods/log
  - services
  - services/status
  - endpoints
  - endpoints/status
  verbs:
  - get
  - list
  - watch
```

** 2. 以 serviceAccount 形式定义一个用户**
```bash
$ kubectl create serviceaccount bmw-svc
$ kubectl  get sa bmw-svc -o yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  creationTimestamp: 2019-05-17T11:26:04Z
  name: bmw-svc
  namespace: default
  resourceVersion: "5587606"
  selfLink: /api/v1/namespaces/default/serviceaccounts/bmw-svc
  uid: 8ee79616-7896-11e9-8bb5-000c29a5444f
secrets:
- name: bmw-svc-token-8f9js
-----------------------------
$ kubectl  get secrets bmw-svc-token-8f9js -o yaml
```
其中 {data.token} 就会是我们的用户 token 的 base64 编码，之后我们配置 kubeconfig 的时候将会用到它.

** 3. 创建角色绑定 **
```yaml
# bmw-svc-rolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: bmw-svc-role
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: bmw-svc-role
subjects:
- kind: ServiceAccount
  name: bmw-svc
  namespace: default
```

** 4. 配置 kubeconfig **

准备好之后呢，就可以用 bmw-svc serviceaccount 访问我们的k8s集群了， 同时我们可以动态更改 Role 或 ClusterRole 的授权来及时控制某个账号的权限(这也是使用 serviceaccount 的好处)；
配置应该如下:
```yaml
...
- context:
    cluster: kubernetes
    user: bmw-svc
  name: bmw-svc
- context:
    cluster: kubernetes
    user: admin
  name: kubernetes
...
users:
- name: admin
  user:
    client-certificate-data: REDACTED
    client-key-data: REDACTED
- name: bmw-svc
  user:
    token: xxxx
```
切换context:
```bash
$ kubectl  config use-context bmw-svc
```
验证：
```bash
$ kubectl  get pod
error: You must be logged in to the server (Unauthorized)
```
说明验证失败，错误的可能性是 secret的token没有进行base64 解码，这个时候也可以切换回去，比如：
```bash
$ kubectl  config use-context kubernetes
```
解码修复之后继续测试:
```bash
$ echo -n 'ZXlKaGJHY2lPaUpTVXpJMU5pSXNJbXRwWkNJNklpSjkuZXlKcGMzTWlPaUpyZFdKbGNtNWxkR1Z6TDNObGNuWnBZMlZoWTJOdmRXNTBJaXdpYTNWaVpYSnVaWFJsY3k1cGJ5OXpaWEoyYVdObFlXTmpiM1Z1ZEM5dVlXMWxjM0JoWTJVaU9pSmtaV1poZFd4MElpd2lhM1ZpWlhKdVpYUmxjeTVwYnk5elpYSjJhV05sWVdOamIzVnVkQzl6WldOeVpYUXVibUZ0WlNJNkltSnRkeTF6ZG1NdGRHOXJaVzR0T0dZNWFuTWlMQ0pyZFdKbGNtNWxkR1Z6TG1sdkwzTmxjblpwWTJWaFkyTnZkVzUwTDNObGNuWnBZMlV0WVdOamIzVnVkQzV1WVcxbElqb2lZbTEzTFhOMll5SXNJbXQxWW1WeWJtVjBaWE11YVc4dmMyVnlkbWxqWldGalkyOTFiblF2YzJWeWRtbGpaUzFoWTJOdmRXNTBMblZwWkNJNklqaGxaVGM1TmpFMkxUYzRPVFl0TVRGbE9TMDRZbUkxTFRBd01HTXlPV0UxTkRRMFppSXNJbk4xWWlJNkluTjVjM1JsYlRwelpYSjJhV05sWVdOamIzVnVkRHBrWldaaGRXeDBPbUp0ZHkxemRtTWlmUS52YmRDNnVEdGFvMDNDWng2ZXNrbGtkNG5UMktQOFI1S3AzOHA3ZUZILVNIWFp6WkRkNG93bUR5cnAzVlF6NkxYeTFfUXRCd25Pc25vQ0lFQ0xPc0YtaXNFVnBaY2k0N1J0UnlzcmtOdTNsMlZkSUFDWGc2dEw4ZVBmZUF1VlJ3US1wUjFueEdlNmdPRU1ybDJKTS1LcmVFNW41Zm9oYVZ3MWVjRzZSendldC1pbnRJdUFaWW1aSi1sdG5sTGFnY1g1NFczOHR3SFdzaDNzd3czN2xSTUNqLVlMVWtXVExabk5rMkNwcWJjNTFvOVFSdDQ1WVUtR3hEX1ctckhlS0JUYWdpdXFjZHdfTDIyQ2UycHJqUGhDSzVuWkk2czJ1dXlkNlFBaUt0dlJHd3RvS205d1J1SEhESzRWellMNjhBdW8zSldncnd5X2VXaXpFRW1CNFF3d3c=' |base64 -d
把解码之后的token进行替换；
$ kubectl  config use-context bmw-svc
$ kubectl  get pod
NAME                 READY   STATUS    RESTARTS   AGE
fortune              2/2     Unknown   0          5d4h
gitrepo-volume-pod   1/1     Unknown   0          5d3h
test-emptydir        1/1     Unknown   0          5d4h
$ kubectl  get secrets
Error from server (Forbidden): secrets is forbidden: User "system:serviceaccount:default:bmw-svc" cannot list resource "secrets" in API group "" in the namespace "default"
```
--------
#### 如何通过客户端证书定义 subjects：
创建 bmw-csr.json 文件：
```json
cat > bmw-csr.json <<EOF
{
  "CN": "bmw",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "k8s",
      "OU": "bmwx4"
    }
  ]
}
EOF
```
生成客户端证书和私钥：
```bash
$ cfssl gencert -ca=/etc/kubernetes/cert/ca.pem \
  -ca-key=/etc/kubernetes/cert/ca-key.pem \
  -config=/etc/kubernetes/cert/ca-config.json \
  -profile=kubernetes bmw-csr.json | cfssljson -bare bmw
$ ls bmw*
bmw.csr  bmw-csr.json  bmw-key.pem  bmw.pem
查看证书内容:
$ cfssl-certinfo -cert bmw.pem
```
创建 bmw 用户的 kubeconfig 文件
```bash
export MASTER_VIP=192.168.10.232
export KUBE_APISERVER="https://${MASTER_VIP}:6443"
------------
kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/cert/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=bmw.kubeconfig
```
设置客户端认证参数
```bash
kubectl config set-credentials bmw \
  --client-certificate=bmw.pem \
  --client-key=bmw-key.pem \
  --embed-certs=true \
  --kubeconfig=bmw.kubeconfig
```
设置上下文参数
```bash
kubectl config set-context bmw \
  --cluster=kubernetes \
  --user=bmw \
  --kubeconfig=bmw.kubeconfig
```
设置默认上下文
```bash
kubectl config use-context bmw --kubeconfig=bmw.kubeconfig
[root@master01 cert]# kubectl get pod --kubeconfig=bmw.kubeconfig     
NAME                 READY   STATUS    RESTARTS   AGE
fortune              2/2     Unknown   0          5d19h
gitrepo-volume-pod   1/1     Unknown   0          5d18h
test-emptydir        1/1     Unknown   0          5d19h
[root@master01 cert]# kubectl get deployments. --kubeconfig=bmw.kubeconfig
Error from server (Forbidden): deployments.extensions is forbidden: User "bmw" cannot list resource "deployments" in API group "extensions" in the namespace "default"
[root@master01 cert]# kubectl get secrets --kubeconfig=bmw.kubeconfig     
Error from server (Forbidden): secrets is forbidden: User "bmw" cannot list resource "secrets" in API group "" in the namespace "default"
```

切回 context :
```bash
[root@master01 cert]# kubectl get secrets --kubeconfig=kubectl.kubeconfig
NAME                  TYPE                                  DATA   AGE
bmw-svc-token-8f9js   kubernetes.io/service-account-token   3      15h
default-token-wzm9h   kubernetes.io/service-account-token   3      62d
tls-secret            kubernetes.io/tls                     2      26d
```

#### 配置访问多集群，其实也是这个道理；
**Tips**   
```
1. 使用--kubeconfig来进行切换集群;
2. 自己实现小工具自动merge kubeconfig文件，然后使用context切换;
```
