# secret
顾名思义， secret 作为 k8s 的一种对象资源，用来存储密钥、token等敏感信息，但是 secret 里的数据部分都是经过 base64 编码的，没有启动实质性安全的作用。

#### 应用 secret
前面介绍ceph的时候，以及在介绍 serviceAccount 的时候， 已经应用过secret了，那么还有一种场景就是，如何从私有镜像仓库下载镜像;

**创建secret:**  
```
# kubectl  create secret  docker-registry dockerhub-secret --docker-username=xxxx --docker-password=xxx --
docker-email=675020908@qq.com

# kubectl  get secrets  dockerhub-secret -o yaml
apiVersion: v1
data:
  .dockerconfigjson: I6eyJodHRwczovL2luZGV4LmRvY2tlci5pby92MS8iOnsidXNlcm5hbWUiOiJibXd4NCIsInBhc3N3b3JkIjoicEA1NWh1YiIsImVtYWlsIjoiNjc1MDIwOTA4QHFxLmNvbSIsImF1dGgiOiJZbTEzZURRNmNFQTWWc9PSJ9fX0=
kind: Secret
metadata:
  creationTimestamp: 2019-05-25T16:55:02Z
  name: dockerhub-secret
  namespace: default
  resourceVersion: "6560105"
  selfLink: /api/v1/namespaces/default/secrets/dockerhub-secret
  uid: d66a8d7a-7f0d-11e9-8e54-000c29a5444f
type: kubernetes.io/dockerconfigjson
```

**在pod中应用 secret**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: private-pod
spec:
  imagePullSecrets:
  - name: dockerhub-secret
  containers:
  - image: username/private:tag
    name: main
```
