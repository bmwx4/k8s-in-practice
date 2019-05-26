# RBAC 与client-go
上个 section 我们已经介绍了 RBAC ，最后我们在功能验证的时候，通过 kubectl 结合使用 kubeconfig 文件来进行测试的，该测试方案也通用试用与client-go这个
库，因为使用 client-go 在构建客户端的时候，是通过kubeconfig文件来配置k8s apiserver信息的，比如:

```go
var kubeconfig string
func init() {
	cobra.OnInitialize(initConfig)
	RootCmd.PersistentFlags().StringVar(&kubeconfig, "kubeconfig", "", "absolute path to the kubeconfig file")
}
{
    config, err := clientcmd.BuildConfigFromFlags("", kubeconfig) //
    if err != nil {
    	panic(err.Error())
    }

    // create the clientset
    clientset, err := kubernetes.NewForConfig(config)
    if err != nil {
    	panic(err.Error())
    }
}
```
kubeconfig 就是 kubernetes 管理员给客户端，或者客户颁发的客户端身份信息，api-server 会根据 kubeconfig 的配置获取用户信息，进行认证、授权和准入控制等；

[测试代码](https://github.com/bmwx4/client-go-kugo)
