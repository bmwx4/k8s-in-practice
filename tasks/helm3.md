# helm3
我们知道，Kubernetes 是一个能够部署和管理容器的平台。然而，在 k8s 里还没有抽象到 “应用” 这一层概念。一个应用往往由多个 k8s 资源 (Deployment、Service、ConfigMap）组成。所以，我们需要一个工具在 k8s 之上来部署和管理一个应用所包含的资源（K8s API Resource），这就是 Helm 所做的事情。
helm3 相较于helm2有较大的架构调整,具体可以参考 [深度解读Helm 3](https://yq.aliyun.com/articles/703943) . 我们下面练习应用一下 helm3 ，并使用阿里云开放的 [App Hub](https://developer.aliyun.com/hub), 如果你还没有理解，你就把它想象成 linux 系统中的yum工具；

#### 安装helm3
```bash
wget https://get.helm.sh/helm-v3.0.0-alpha.1-darwin-amd64.tar.gz
tar xvf helm-v3.0.0-alpha.1-darwin-amd64.tar.gz
cp helm /usr/local/bin/
```

#### 测试helm3
需要具备一个 kubernetes 环境，我这边使用 minikube 来快速启动一个k8s集群, 由于需要下载官方镜像，所以走了我自己的代理，大家可以根据自身环境调整:
```bash
# https_proxy=192.168.10.153:1087  minikube start --docker-env HTTP_PROXY=192.168.10.153:1087 --docker-env HTTPS_PROXY=192.168.10.153:1087 --docker-env NO_PROXY=192.168.99.0/24
也可以使用国内的镜像仓库:
# minikube start --registry-mirror=https://registry.docker-cn.com
```
k8s 启动完成之后，就可以玩helm了，首先需要在k8s集群中初始化helm:
```bash
helm init
```
初始化之后， 首先可以看看从哪里能下载应用呢?
```bash
# helm repo list
NAME  	URL
stable	https://kubernetes-charts.storage.googleapis.com
local 	http://127.0.0.1:8879/charts
```
我们没有local的， 官方的又被xx了， 幸好有雷锋,下面我们添加一下app hub:
```bash
# helm repo add apphub https://apphub.aliyuncs.com
# helm repo list
NAME  	URL
stable	https://kubernetes-charts.storage.googleapis.com
local 	http://127.0.0.1:8879/charts
apphub	https://apphub.aliyuncs.com
```
下面你比如你想run一个wordpress,那我们可以像使用yum一样，先搜索一下:
```bash
# helm search wordpress
NAME            	CHART VERSION	APP VERSION	DESCRIPTION
apphub/wordpress	5.12.3       	5.2.1      	Web publishing platform for building blogs and ...
stable/wordpress	6.0.0        	5.2.2      	Web publishing platform for building blogs and ...

guestbook 也是相同搜索方式:
# helm search guestbook
NAME                   	CHART VERSION	APP VERSION	DESCRIPTION
apphub/guestbook       	0.2.0        	           	A Helm chart to deploy Guestbook three tier web...
apphub/guestbook-kruise	0.3.0        	           	A Helm chart to deploy Guestbook three tier web...
```
有两条记录，一个是 apphub 阿里云源里的，一个是 stable 官方的，下面我们使用阿里云提供的源来安装我们的 wordpress:
```bash
helm install wordpress apphub/wordpress --set service.type=NodePort
NAME: wordpress
LAST DEPLOYED: 2019-07-21 20:48:43.766862 +0800 CST m=+1.449421235
NAMESPACE: default
STATUS: deployed

NOTES:
1. Get the WordPress URL:

  export NODE_PORT=$(kubectl get --namespace default -o jsonpath="{.spec.ports[0].nodePort}" services wordpress-wordpress)
  export NODE_IP=$(kubectl get nodes --namespace default -o jsonpath="{.items[0].status.addresses[0].address}")
  echo "WordPress URL: http://$NODE_IP:$NODE_PORT/"
  echo "WordPress Admin URL: http://$NODE_IP:$NODE_PORT/admin"

2. Login with the following credentials to see your blog

  echo Username: user
  echo Password: $(kubectl get secret --namespace default wordpress-wordpress -o jsonpath="{.data.wordpress-password}" | base64 --decode)
```
查看pod是否创建成功:
```bash
# kubectl get pod
wordpress-mariadb-0                    1/1     Running   0          9m40s
wordpress-wordpress-56d9ddd46d-qmwj2   1/1     Running   0          9m35s
# kubectl get svc
wordpress-mariadb     ClusterIP   10.104.207.111   <none>        3306/TCP                     19m
wordpress-wordpress   NodePort    10.101.24.4      <none>        80:32714/TCP,443:32713/TCP   19m
```

按照提示， 获取访问 wordpress 的url进行访问，我的环境访问这个url 访问不通，不知道是否和我的代理有关系，因为创建应用时，我们指定了node port访问方式，所以我可以通过minikube ssh 登录到vm，查看下能访问到的br ip，通过这个ip访问 :
```
# minikube ssh
                         _             _
            _         _ ( )           ( )
  ___ ___  (_)  ___  (_)| |/')  _   _ | |_      __
/' _ ` _ `\| |/' _ `\| || , <  ( ) ( )| '_`\  /'__`\
| ( ) ( ) || || ( ) || || |\`\ | (_) || |_) )(  ___/
(_) (_) (_)(_)(_) (_)(_)(_) (_)`\___/'(_,__/'`\____)

$ ifconfig eth1
eth1      Link encap:Ethernet  HWaddr 08:00:27:1F:A1:7A
          inet addr:192.168.99.100  Bcast:192.168.99.255  
```
所以也可以通过 192.168.99.100 这个ip 来访问我们的服务:
![wordpress](/images/wordpress.png)

***就是这么方便！！！***
