# ReplicaSet
新一代的 RC，与RC的行为是一直的，但是 pod 选择器的表达能力更强；我们简称"RS"。

### 创建RS
把在RC中删除的RC，使用RS进行替换，看是否可以正常匹配pod：
```yaml
# kubia-rs.yaml
apiVersion: apps/v1beta2
kind: ReplicaSet
metadata:
  name: kubia
spec:
  replicas: 3
  selector:
    matchLabels:
      app: kubia
  template:
    metadata:
      labels:
        app: kubia
    spec:
      containers:
      - name: kubia
        image: luksa/kubia
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 8080
```
除了apiVersion和kind的不同, 最大的区别就是 selector ；RS 需要直接在 selector 中指定标签，而是通过 matchLabels 来指定它们；
```bash
kubectl create -f kubia-rs.yaml
kubectl describe rs kubia
```
由于副本数和当前匹配的pod数相同，所以不会新建pod或者删除pod

#### 标签选择器表达式用法
RS相较于RC最大的改进就是 标签选择器更灵活了，比如RS支持标签选择器使用 matchExpressions 属性来扩展选择器，比如：
``` yaml
apiVersion: apps/v1beta2
kind: ReplicaSet
metadata:
  name: kubia
spec:
  replicas: 3
  selector:
    matchExpressions:
      - key: app
        operator: In
        values:
         - kubia
  template:
    metadata:
      labels:
        app: kubia
    spec:
      containers:
      - name: kubia
        image: luksa/kubia
        imagePullPolicy: IfNotPresent
```
可以给选择器增加多个表达式，每个表达式都必须包含一个key，一个 operator，但是可能包含多个 values，或者没有 values，取决于operator的值；比如：
```
In: Label的值必须与其中的一个value匹配
NotIn: Label 的值与任何指定的values 不匹配
Exists: pod 必须包含一个指定名称的标签(值不重要),使用才运算符时，无需指定values
DoesNotExists: pod 不得包含有指定名称的标签。values属性不得指定
```
如果指定了多个表达式的话，这些表达式的结果都必须为true，才能使pod与RS选择器进行匹配，如果同时指定 matchExpressions 和 matchLabels，那么所有标签都必须匹配的同时，且所有表达式必须为true，以使pod与RS选择器匹配。

### 删除RS
```bash
$ kubectl delete rs kubia
```
### 总结
RS是RC的替代品， 生产环境中也不会有人去创建一个RC，除非是很早期的用户。
