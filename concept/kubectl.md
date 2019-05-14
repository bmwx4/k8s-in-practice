
#### Basic Commands：

| 命令 | 说明 | 举例 |
| :------| ------: | :------: |
| create | 从文件或stdin创建资源 | 稍微长一点的文本 |
| run    | xxxxx | xxxxxx |
| expose |  为deployment，pod创建Service | 中等文本 |
| get    | 最基本的查询命令 | kubectl get depod/pod |
| explain| 查看资源定义  | kubectl explain pod |
| edit   | 使用系统编辑器编辑资源 | kubectl edit deploy/nginx|
| delete | 删除指定资源，比如:文件名、资源名、label selector|kubectl delete po -l foo=bar|

-----
#### deploy Commands:
| 命令 | 说明 | 举例 |
| :------| ------: | :------: |
|rollout| Deployment, Daemonset的升级过程管理,查看状态、操作历史、暂停升级、恢复升级、回滚等 | xx |
| rolling-update | 客户端滚动升级，仅限ReplicationController| xx |
| scale |  修改Deployment, ReplicaSet, ReplicationController, Job的实例数 | xx |
| autoscale | 为Deploy, RS, RC配置自动伸缩规则(依赖heapster和hpa) | xx |

-----
#### Cluster Management Commands:
| 命令 | 说明 | 举例 |
| :------| ------: | :------: |
| certificate | Modify certificate resources. | xx |
| cluster-info | 查看集群信息 | xx |
| top | 查看资源占用率(依赖 heapster) |xx |

-----
#### Troubleshooting and Debugging Commands:
| 命令 | 说明 | 举例 |
| :------| ------: | :------: |
| describe | 查看资源详情 | xx |
| logs | 查看pod内容器的日志 | xx |
| attach | Attach到pod内的一个容器 | xx |
| exec | 在指定容器内执行命令 | xx |
| port-forward | 为pod创建本地端口映射 | kubectl  port-forward httpod 8888:80 |
| proxy | 为Kubernetes API server创建代理 | kubectl proxy --port=8080 --address="0.0.0.0" |
| cp | 容器内外/容器间文件拷贝 | xx |

------
#### Advanced Commands:
| 命令 | 说明 | 举例 |
| :------| ------: | :------: |
|apply  | 从文件或stdin创建/更新资源  | xx |
|patch  | 使用strategic merge patch语法更新对象的某些字段 | xx |
|replace | 从文件或stdin更新资源 | xx |
|convert | 在不同API版本之间转换对象定义 | xx |

-----
#### Settings Commands:
| 命令 | 说明 | 举例 |
| :------| ------: | :------: |
| cordon |  标记节点(node)为unschedulable | xx |
|uncordon | 标记节点为schedulable | xx |
|drain |驱逐节点上的应用，准备下线维护 | xx |
|taint | 修改节点taint标记 | xx |
| label | 给资源设置label | xx |
| annotate | 给资源设置annotation | xx |
| completion | 获取shell自动补全脚本(支持bash和zsh)

----- 
#### Other Commands
| 命令 | 说明 | 举例 |
| :------| ------: | :------: |
| api-versions | 以"group/version"的形式输出支持的api version| xx |
| config | 修改kubectl配置(kubeconfig文件)，如context | xx |
|version | 查看客户端和Server端K8S版本 | xx |
| help | Help about any command | xx |

#### 实用技巧
kubectl命令太多太长记不住?
查看资源缩写
```
$ kubectl describe
```
配置kubectl自动补全  
```bash
$ source <(kubectl completion bash)
$ source <(kubectl completion zsh)
```
kubectl写yaml太累，找样例太麻烦? 用run命令生成  
```bash
$ kubectl run --image=nginx my-deploy -o yaml --dry-run > my-deploy.yamls
```
用get命令导出yaml  
```bash
$ kubectl get statefulset/nginx -o=yaml --export > nginx.yaml
```

Pod亲和性下面字段的拼写忘记了  
```bash
kubectl explain pod.spec.affinity.podAffinity
```
快速创建pod：
```bash
# kubectl  run busybox  --image=busybox --image-pull-policy=IfNotPresent --generator=run-pod/v1 --command -- sleep 1000
```

![kubectl](../images/kubernetes-kubectl-cheatsheet.png)
