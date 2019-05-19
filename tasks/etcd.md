# etcd 优化

#### 背景：
由于线上有些k8s集群的etcd流量访问不均匀，就会导致etcd那边经常会收到慢查询，如果客户端使用不当的话，可能导致把当前承受etcd压力的member压垮：监控结果显示如下：
![etcd-monitor](/images/etcd-monitor.png)
#### 原因分析：
因为etcd客户端配置是把所有member有序的配置上的，比如：apiserver的配置:
```bash
kube-master00: KUBE_ETCD_SERVERS="--etcd-servers=https://etcd00:2379,https://etcd01:2379,https://etcd02:2379,https://etcd03:2379,https://etcd04:2379"
kube-master01: KUBE_ETCD_SERVERS="--etcd-servers=https://etcd0:2379,https://etcd01:2379,https://etcd02:2379,https://etcd03:2379,https://etcd04:2379"
```
因为我们一共两台apiserver， 我们希望两台 apiserver 分别把写请求定向到两台etcd member中去，比如下面的情况:
![etcd-ok](/images/etcd-ok.png)
上面是看上去正常的case，这也是随机的，所以我想了一个土办法，就是先手动修改apiserver的配置，把两台apiserver 中etcd 配置的顺序打乱，强制apiserver把写请求定向到不同的etcd member中去。虽然有可能也会碰撞到一起，但是概率会很小；
#### 解决方案:
更新其中一台apiserver的etcd配置，重启apiserver；
#### 验证结果：
修改完配置之后，等待一会，查看监控如下，写请求打到不通的member中了，符合预期；
![etcd-fixed](/images/etcd-fixed.png)
