https://draveness.me/understanding-kubernetes/
https://draveness.me/kubernetes-pod/

# 设计理念
- Declarative 申明式
    - 实现手段：YAML描述终态、状态监控、状态转移
- Explicit API 显示接口
- Non-Invasive 无侵入
- Potablility 可移植



# 架构: 中心化的client-master-worker

## client
RESTful接口，kubectl封装为命令行。

## master
- API Server
- Controller
- Scheduler
- etcd

## worker
- kubelet 主内
- kubeproxy 主外
- Pod
- Container

### Pod
https://kubernetes.io/docs/concepts/workloads/pods/

Pod内部有两种不同的container：
- Init Container  
- Regular Container 

通过 CGroups 机制实现Pod内Container之间共享网络、存储？


kubelet管理Pod的生命周期：Create -> Probe -> Running -> Shutdown -> Restart

### 网络共享和隔离原理

NAT桥接模式：
veth 虚拟网卡
iptables 路由转发


### CRI(Container Runtime Interface)
Docker提供了整套从构建到运行的开发者工具，开发者可以很快上手Docker，并在本地运行并管理Docker容器。   
然而，在集群中运行的容器运行时往往不需要那么复杂的功能，K8s只需要CRI定义的那些功能。


依赖接口而非实现，依赖反转了，K8s可以不受容器实现差异的困扰了。

容器实现：CRI-O, Containerd, rkt, Docker container


### Service


kube-ctl
- service controller
- endpoint controller


kube-proxy 
- userspace
- iptable
- ipvs





