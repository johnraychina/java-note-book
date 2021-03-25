
# K8s组件
## controll plane components

- kube-apiserver
- etcd
- kube-scheduler
- kube-controller-manager
    - node controller
    - job controller
    - endpoint controller
    - service account & token controller
- cloud-controller-manager: 
node controller, route controller, service controller可以依赖cloud provider.



## node components
- kubelet
- kube-proxy
- container runtime

## addons
cluster-level features, namespace resources for addons belong withing the kube-system namespace.
- DNS(必选)
- WebUI(DashBoard)
- Container Resource Monitoring
- Cluster-level Logging

# 使用k8s组件

## k8s组件管理

三种方式：
1. 命令行
```sh
kubectl create deployment nginx --image nginx
```
2. 命令式配置
```sh
kubectl create deployment nginx --image nginx
```

3. 申明式配置
```sh
kubectl diff -f configs/
kubectl apply -f configs/
```



## API groups and versioning
API groups that can be enabled or disabled.

## Object Names and IDs
ID是k8s自动生成的集群内唯一的，下面这三种命名有一些约束条件：
DNS Subdomain Names：最多253字符，只能是[小写字母数字, '-' or '.']，起始结束都只能是小写字母数字。   
DNS Label Names：最多63字符，只能是[小写字母数字, '-' or '.']，起始结束都只能是小写字母数字。   
Path Segment Names: the name may not be "." or ".." and the name may not contain "/" or "%".   

## Namespaces

DNS Entry全名: <service-name>.<namespace-name>.svc.cluster.local   
如果容器只用service-name，默认用当前namespace, 如果你希望跨namespace引用DNS名，得用全名。   


NOT all objects are in Namespace   
大部分k8s资源（pod, service, replication controller, and others)在namespace内。   
但是namespace资源本身，以及底层资源（node, persistentVolume)不在任何namespace中。   

你可以用下面的命令查看哪些资源在namespace内（受它管理）：
```sh
# In a namespace
kubectl api-resources --namespaced=true

# Not in a namespace
kubectl api-resources --namespaced=false
```

## Labels and Selectors
首先给组件打标(metadata-labels): 
```yaml
kind: Pod
metadata:
  name: label-demo
  labels:
    environment: production
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.14.2
    ports:
    - containerPort: 80

```


然后在命令行做筛选:
```sh
kubectl get pods -l environment=production,tier=frontend
```


或者yaml引用组件时做筛选：
```yaml
selector:
    component: redis
```


```yaml
selector:
  matchLabels:
    component: redis
  matchExpressions:
    - {key: tier, operator: In, values: [cache]}
    - {key: environment, operator: NotIn, values: [dev]}
```




## Annotations
你可以使用metadata-annotations，将一些与身份无关的元数据关联到组件上，以便让客户端能取到这些元数据：
可选的前缀，用"/"分隔。
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: annotations-demo
  annotations:
    imageregistry: "https://hub.docker.com/"
spec:
  containers:
  - name: nginx
    image: nginx:1.14.2
    ports:
    - containerPort: 80
```

## Field Selectors
```sh
kubectl get pods --field-selector status.phase=Running
```

- metadata.name=my-service
- metadata.namespace!=default
- status.phase=Pending

## Recommended Labels
除了kubectl命令行和dashboard之外，你还可以对k8s组件做可视化的管理，
定义一组所有工具都能识别的通用标签(prefix "app.kubernetes.io")，可以提高互操作性。

尽管metadata是按照application的概念组织的，但k8s并不是一个PaaS，所以并不强制要求一个正式的application概念。
而是通过metadata不那么正式的方式来描述application概念，它包含什么也不做强制要求。

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  labels:
    app.kubernetes.io/name: mysql
    app.kubernetes.io/instance: mysql-abcxzy
    app.kubernetes.io/version: "5.7.21"
    app.kubernetes.io/component: database
    app.kubernetes.io/part-of: wordpress
    app.kubernetes.io/managed-by: helm
```





