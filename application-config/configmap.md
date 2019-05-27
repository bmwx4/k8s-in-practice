# ConfigMap
为了能够让业务应用与配置进行解耦，并且不影响在运行中的 pod， k8s 提供了 ConfigMap 资源对象，帮我们完成业务与配置间的解耦，使用 valueFrom 字段替代写死的 value 字段， ConfigMap 成为了环境变量或者配置的来源;

#### 创建 ConfigMap
有如下几种方式 通过 kubectl 创建 ConfigMap，比如:
```bash
kubectl create configmap my-config
  --from-file=foo.json            # 文件
  --from-file=config-ops/        # 文件件
  --from-literal=key=value      # 字面量
```
比如我们使用 字面量的形式创建一个configMap
```bash
kubectl create configmap fortune-config --from-literal=sleep-interval=25
```

使用yaml 方式创建一个 ConfigMap:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fortune-config
data:
  sleep-interval: "25"
```

#### 应用 ConfigMap
在容器中使用 configmap:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: fortune-args-from-configmap
spec:
  containers:
  - image: luksa/fortune:args
    imagePullPolicy: IfNotPresent
    env:                  #定义环境变量
    - name: INTERVAL
      valueFrom:
        configMapKeyRef:
          #optional: true    
          name: fortune-config
          key: sleep-interval
    args: ["$(INTERVAL)"]
    name: html-generator
    volumeMounts:
    - name: html
      mountPath: /var/htdocs
  - image: nginx:alpine
    name: web-server
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html
      readOnly: true
    ports:
    - containerPort: 80
      protocol: TCP
  volumes:
  - name: html
    emptyDir: {}
```
"optional: true" 说明即使应用的 configMap 不存在，也不会报错, 正常情况下，业务的程序配置都会有自己的默认值;

#### 包含来源于 configMap 中的所有环境变量
如果 configMap 中包含有大量的条目，那么可以把这些环境变量一并导入到容器,比如:
```yaml
spec:
  containers:
    envFrom:                  #不是使用env
    - prefix: CONFIG_         # 环境变量名的前缀
      configMapKeyRef:
        #optional: true    
        name: fortune-config
    args: ["$(INTERVAL)"]
```
