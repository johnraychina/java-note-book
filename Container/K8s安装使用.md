
# 参考资料
[Nana视频教程](https://www.youtube.com/watch?v=X48VuDVv0do)
[jimmysong的k8s手册](https://jimmysong.io/kubernetes-handbook/#)
[k8s官方教程](https://kubernetes.io/docs/tutorials/kubernetes-basics/)

# 笔记

## k8s是什么
**背景**
随着应用越来越复杂，微服务越来越流行，
Linux 提供了容器所需的基础技术，
Docker 提供了容器能力（镜像打包、分发、容器运行）    

后来，K8s 提供容器管理能力：   
- 服务发现和负载均衡
- 存储管理
- 面向终态的资源管理：drive current state towards the provided desire state
- 资源管理自动最优化：optimize(资源, 需求)
- 自愈能力
- 秘钥和配置管理

k8s提供了基础模块 + 插件机制，可以扩展功能。


## K8s Main Components
Node:
    physical server
Pod:
    Smallest unit of k8s
    Abstraction over container
    Usually 1 applicatio per Pod
    Each Pod get its own IP address
    New IP address on re-creation

Service:
    permanent IP address
    lifecycle of Pod and Service NOT connected
    2个主要作用：永久IP，负载均衡

    Ingress 转发
    Internal service

ConfigMap 配置 :
    external configuration of your application

Secret 秘钥:
    used to store secret data
    base64 encoded

Volume 卷:
    K8s doesn't manage data persistence

Deployment 部署:
    blueprint for creating pods
    you create Deployments
    abstraction of pods

StatefulSet 状态集    
    DB can't be replicated via Deployment!!
    Deplying StatefulSet not easy!! 一致性问题
    DB are often hosted out of K8s cluster


## K8s Architecture

Worker machine in K8s cluster:
    Worker Nodes do the actual work
    each node has multiple Pods on it 
    3 processes must be installed on every Node
        Container runtime
        Kubelet: 
            Kubelet interacts with both the container and node
            Kubelet starts the pod with a container inside
        Kube Proxy: forwards the requests 


Q: How do you interact with this cluster? How to:
- schedule pod?
- monistor
- re-schedule/restart pod?

A: Managing processes are done by Master Nodes

Master processes:
    API Server: 
        cluster gateway
        acts as a gatekeeper for authentication!
    Scheduler: where to put the Pod?
    Controller manager: detects cluster state changes
    etcd: 
        acts as the cluster brain
        cluster changes get stored in the key-value store
        application data is NOT stored in etcd!


Minimum cluster setting:
2 Master Nodes
3 Workder Nodes

## minikube 和 kubectl 安装
什么是minikube：
creates Virtual Box on your laptop
Node runs in that Virtual Box
1 Node K8s cluster
for testing purposes

command line tool for K8s cluster：
Minikube CLI: for start up/deleting the cluster
Kubectl CLI: for configuring the Minikube cluster


```sh
brew update
# 安装虚拟化
brew install hyperkit

# 安装minikube，会依赖安装kubectl
brew install minikube

# 使用虚拟化工具启动minikube
minikube start --vm-driver=hyperkit

# 查看minikube状态
minikube status
kubectl get nodes
kubectl version
```

**kubectl --help**
看帮助文档是学习一个命令的好办法，从这里你可以看到作者的设计原则，知道作者是怎么想的，能帮助你更好地理解和使用他开发的工具。



## kubectl 部署
```sh
# 查询节点、服务、pod、复制集、部署
kubectl get nodes/services/pod/replicaset/deployment

# 创建和编辑deployment
kubectl create deployment [name] --image=image
kubectl edit deployment [name]

# 查看一下pod的运行状态 
kubectl get pod
```


## kubectl 调试pods
```sh
# 日志
kubectl logs [pod name]
kubectl describe pod [pod name]

# 进入pod命令行：
kubectl exec - it [pod name] -- bin/bash

# 删除
kubectl delete deployment [name]

# 参数配置
kubectl apply -f [config file]
```


## YAML部署文件

#### deployment
使用命令行创建deployment时，用的是默认配置，你当然可以通过命令行指定很多参数，
你的命令行可能会非常冗长，所以我们需要YAML文件：

part1: metadata
part2: specification
part3: status (automatically generated and added by K8s)

service.selector--> deployments.selector

#### service


## 使用kubectl部署ngix


## 使用kubectl部署dubbo
kubectl create deployment dubbo-admin-depl --image=apache/dubbo-admin

## 创建一个由mongdb的网站应用
- deployment
    - configmap: database url
    - secret: database user + password

- service
    - internal service: mongodb
    - external service: mongo-express


1. 拉取[公共仓库mongodb镜像](https://hub.docker.com/_/mongo)进行部署。      
文档有说明默认监听端口是 27017
```sh
kubectl create deployment mongodb --image=mongo
```

2. 通过-o 输出deployment yaml文件   
```sh
kubectl get deployment mongodb -o yaml >> mongodb.yaml
```

3. 设置 mongodb deployment  yaml文件   
首先删掉status和一些无关的设置

设置端口

设置环境变量（官方文档上有变量名）   
MONGO_INITDB_ROOT_USERNAME, MONGO_INITDB_ROOT_PASSWORD

密码设置   
为了安全起见，创建密码配置mongodb-secret.yaml
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mongodb-secret
type: Opaque
data:
  mongo-root-username: dGVzdA==
  mongo-root-password: MTIz
```
说明：用户名和密码用base64（作为示例，简单起见用base64）加密
echo -n 'test' | base64 得到 dGVzdA==   
echo -n '123' | base64 得到MTIz   


```sh
# 将密码保存到k8s存储中(etcd)
kubectl apply -f mongodb-secret.yaml

# 检查一下
kubectl get secret
```

修改mongodb.yaml中的env
把用户名和密码改为引用前面存储到k8s的Secret:
```yaml
        env:
        - name: MONGO_INITDB_ROOT_USERNAME
          valueFrom: 
            secretKeyRef:
              key: mongo-root-username
              name: mongodb-secret
        - name: MONGO_INITDB_ROOT_PASSWORD
          valueFrom: 
            secretKeyRef:
              key: mongo-root-password
              name: mongodb-secret
```


4. 设置mongodb数据库的service yaml
service和deployment可以放一起，用 --- 隔开就行

- kind:             "Service"
- metadata/name:    a random name
- selector:         to connect to Pod through label
- ports:            
    - port:         Service port
    - targetPort:   containerPort of Deployment


5. 设置mongo-express网站deployment yaml，[官方镜像](https://hub.docker.com/_/mongo-express)   
参照文档：
- env中设置mongodb连接(valueFrom-configMapKeyRef)
- env中设置mongodb用户名密码(valueFrom-secretKeyRef)

6. 设置mongo-express网站service yaml：   
暴露为一个外部服务：外部端口、内部端口
- type:         "LoadBalancer"
- nodePort:     external port(30000~32767)

```sh
# apply yaml配置
kubectl apply -f mongo-express.yaml

# 查看一下服务
# 看到mongo-express-service服务的外部IP是 pending
kubectl get service
```

```sh
# 分配外部IP给这个服务
minikube service mongo-express-service
```
7. 自动或手工打开网页
http://[对外IP地址]:[nodePort]/


## Namespace






# 笔记

面向过程 VS 面向终态的调度

安全容器
Kata Containers
Firecracker
gVisor

ebpf
ptrace

etcd, Istio, Envoy

kube-apiserver

controller

kubelet




