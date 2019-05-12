
# client-go

Kubernetes官方从2016年8月份开始，将Kubernetes资源操作相关的核心源码抽取出来，独立出来一个项目Client-go，作为官方提供的Go client。Kubernetes的部分代码也是基于这个client实现的，所以对这个client的质量、性能等方面还是非常有信心的。
client-go是一个调用kubernetes集群资源对象API的客户端，即通过client-go实现对kubernetes集群中资源对象（包括deployment、service、ingress、replicaSet、pod、namespace、node等）的增删改查等操作。大部分对kubernetes进行前置API封装的二次开发都通过client-go这个第三方包来实现。

#### 主要package
主要的几个package包的功能说明：
```
kubernetes： 访问 Kubernetes API的一系列的clientset
discovery：通过Kubernetes API 进行服务发现
dynamic：对任意Kubernetes对象执行通用操作的动态client
transport：启动连接和鉴权auth
tools/cache：controllers控制器
```
#### Informer
Client-go包中一个相对较为高端的设计在于Informer的设计，我们知道我们可以直接通过Kubernetes API交互，但是考虑一点就是交互的形式，Informer设计为List/Watch的方式。Informer在初始化的时先通过List去从Kubernetes API中取出资源的全部object对象，并同时缓存，然后后面通过Watch的机制去监控资源，这样的话，通过Informer及其缓存，我们就可以直接和Informer交互而不是每次都和Kubernetes API交互。
Informer另外一块内容在于提供了事件handler机制，并会触发回调，这样上层应用如Controller就可以基于回调处理具体业务逻辑。因为Informer通过List、Watch机制可以监控到所有资源的所有事件，因此只要给Informer添加ResourceEventHandler 实例的回调函数实例取实现OnAdd(obj interface{}) OnUpdate(oldObj, newObj interface{}) 和 OnDelete(obj interface{})这三个方法，就可以处理好资源的创建、更新和删除操作
Kubernetes中都是各种controller的实现，各种controller都会用到Informer。
![client-go-controller-interaction](/images/client-go-controller-interaction.jpeg)

### 常用apis
#### deployment
```go
//声明deployment对象
var deployment *v1beta1.Deployment
//构造deployment对象
//创建deployment
deployment, err := clientset.AppsV1beta1().Deployments(<namespace>).Create(<deployment>)
//更新deployment
deployment, err := clientset.AppsV1beta1().Deployments(<namespace>).Update(<deployment>)
//删除deployment
err := clientset.AppsV1beta1().Deployments(<namespace>).Delete(<deployment.Name>, &meta_v1.DeleteOptions{})
//查询deployment
deployment, err := clientset.AppsV1beta1().Deployments(<namespace>).Get(<deployment.Name>, meta_v1.GetOptions{})
//列出deployment
deploymentList, err := clientset.AppsV1beta1().Deployments(<namespace>).List(&meta_v1.ListOptions{})
//watch deployment
watchInterface, err := clientset.AppsV1beta1().Deployments(<namespace>).Watch(&meta_v1.ListOptions{})
```

------
#### service
```go
//声明service对象
var service *v1.Service
//构造service对象
//创建service
service, err := clientset.CoreV1().Services(<namespace>).Create(<service>)
//更新service
service, err := clientset.CoreV1().Services(<namespace>).Update(<service>)
//删除service
err := clientset.CoreV1().Services(<namespace>).Delete(<service.Name>, &meta_v1.DeleteOptions{})
//查询service
service, err := clientset.CoreV1().Services(<namespace>).Get(<service.Name>, meta_v1.GetOptions{})
//列出service
serviceList, err := clientset.CoreV1().Services(<namespace>).List(&meta_v1.ListOptions{})
//watch service
watchInterface, err := clientset.CoreV1().Services(<namespace>).Watch(&meta_v1.ListOptions{})
```

-----
#### pod
```go
//声明pod对象
var pod *v1.Pod
//创建pod
pod, err := clientset.CoreV1().Pods(<namespace>).Create(<pod>)
//更新pod
pod, err := clientset.CoreV1().Pods(<namespace>).Update(<pod>)
//删除pod
err := clientset.CoreV1().Pods(<namespace>).Delete(<pod.Name>, &meta_v1.DeleteOptions{})
//查询pod
pod, err := clientset.CoreV1().Pods(<namespace>).Get(<pod.Name>, meta_v1.GetOptions{})
//列出pod
podList, err := clientset.CoreV1().Pods(<namespace>).List(&meta_v1.ListOptions{})
//watch pod
watchInterface, err := clientset.CoreV1().Pods(<namespace>).Watch(&meta_v1.ListOptions{})
```

------
[client-go](https://github.com/kubernetes/client-go)  
[client-go测试代码](https://github.com/bmwx4/client-go-kugo)
