# kubectl 常用姿势

- 获取某个 node 上的 pods列表
```bash
 kubectl get pod -o wide --field-selector spec.nodeName={$nodename}
```
