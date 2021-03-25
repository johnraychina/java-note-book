
根据集群类型，一个Node可能是一个虚拟机或者物理机。

每个Node都由control plane管理，并包含了一些运行Pod的必要服务。
Node里面包括kubelet, kube-proxy, container runtime.

## Management
There are two main ways to add Nodes to the API server:
1. kubelet on a node self-registers to the control plane:
--register-node=true

2. You, or another human user, mannually add a Node obejct:
--register-node=false

The name of a Node object must be a valid DNS subdomain name.


```json
{
  "kind": "Node",
  "apiVersion": "v1",
  "metadata": {
    "name": "10.240.79.157",
    "labels": {
      "name": "my-first-k8s-node"
    }
  }
}
```

## Node status
```sh
kubectl describe node <insert-node-name-here>
```
- addresses
- conditions
- capacity and allocatable
- info
 








