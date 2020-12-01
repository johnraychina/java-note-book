
# 参考资料
https://www.youtube.com/watch?v=X48VuDVv0do
https://jimmysong.io/kubernetes-handbook/#


# 笔记

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

## kubectl 部署
```sh
kubectl get nodes/services/pod/replicaset/deployment
# 创建和编辑deployment
kubectl create deployment [name] --image=image
kubectl edit deployment [name]
```


## kubectl 调试pods
```sh
# 日志
kubectl log [pod name]
kubectl describe pod [pod name]
# 进入pod命令行：
kubectl exec - it [pod name] -- bin/bash

# 删除
kubectl delete deployment [name]

# 参数配置
kubectl apply -f [config file]
```
## yaml deployment configuration file
part1: metadata
part2: specification
part3: status (automatically generated and added by K8s)


service.selector--> deployments.selector


## 使用kubectl部署ngix


## 使用kubectl部署dubbo
kubectl create deployment dubbo-admin --image=apache/dubbo-admin





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

