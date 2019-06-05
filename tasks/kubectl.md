# kubectl 常用姿势

| 需求 | 命令 | 备注 |
|:-----|:------|:------|
| 获取某个 node 上的 pods列表 | kubectl get pod -o wide --field-selector spec.nodeName=${nodename} | 依赖 --field-selector |
| 更新 pod/node 的 annotations | kubectl annotate pod ${podName} ${podAnnontations}=${value} --overwirte | 如果已存在 Annontations，通过 --overwirte 选项覆盖|
| 更新 pod/node 的 labels | kubectl label pod ${podName} ${podLabel}=${value} --overwirte |如果已存在该 label ，通过 --overwirte 选项覆盖|
|获取指定 kubelet 版本的 node 名称| bash kubectl get no -o json &#124; jq -r '.items[] &#124;  select(.status.nodeInfo.kubeletVersion=="v1.12.4-41+1a17526c9ecfae") &#124; .metadata.name' | Reserved |
