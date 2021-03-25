
**workload**
A workload is an application running on kubernetes.


K8s provides serveral build-in workload resources:
- Deployment and ReplicaSet
- StatefulSet
- DeamonSet
- Job & CronJob

**Pods**
Pods are the smallest deployable units of computing that you can create and 
manage in kubernetes.

In terms of Docker concepts, a Pod is similar to a group of Docker containers with shared namespaces and shared filesystem volumes.


**PodTemplate**
Controllers for workload resources create Pods from a pod template and 
manage those Pod on your behalf.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: hello
spec:
  template:
    # This is the pod template
    spec:
      containers:
      - name: hello
        image: busybox
        command: ['sh', '-c', 'echo "Hello, Kubernetes!" && sleep 3600']
      restartPolicy: OnFailure
    # The pod template ends here
```


### Resource sharing and communication
Pods enable resource sharing and communication among their 
constituent containers.

- Storage: 
- Networking: 
- share an ip address and port space, localhost
- IPC
- Shared Memory

**Privileged mode for containers**
特权模式：在container spec中指定privileged标志。

**Static pod**
自主管理，走kubelet管理，不受control plane管理。


### Pod LifeCycle
Pod phase: Pending, Running, Succeeded, Failed, Unknown   

Container states: Waiting, Running, Terminating   

Container restart policy:  spec-restartPolicy   

Termination
TERM/STOPSIGNAL + graceful shutdown period(default 30s)
when grace period has expired the KILL signal is sent.

Forced Pod termination: 默认优雅停机30s, 修改时间
```sh
kubectl delete xxx --grace-period=0 
```


### Init Containers

- Init containers always run to completion.
- Each init container must complete successfully before the next one starts.

### Pod Topology Spread Constraints
spec-topologySpreadConstraints

### Disruptions

### Ephemeral Conatainers
In most cases, we use ephemeral containers to inspect services rather than to build applications.

kind: EphemeralContainers

### Workload Resources
- Deployment
- ReplicaSet
- StatefulSet
- DeamonSet
- Jobs
- Garbage Collection
- TTL Controller for Finished Resources
- CronJob
- ReplicationController


#### Use Case: Creating a Deployment
kubectl apply -f https://k8s.io/examples/controllers/nginx-deployment.yaml
kubectl get deployments
kubectl rollout status deployment/nginx-deployment.
kubectl get deployments
kubectl get rs
kubectl get pods --show-labels