
# 集群部署
http://spark.apache.org/docs/latest/cluster-overview.html

[集群部署图](spark-cluster-overview.png)

## 基本组件
- Driver Prgram(SparkContext Spark session)
- Cluster Manager: Standalone/Mesos/Yarn/K8s 
- WorkerNode(Executor, Cache)

- Executor	
A process launched for an application on a worker node, that runs tasks and keeps data in memory or disk storage across them. Each application has its own executors.

- Task	
A unit of work that will be sent to one executor

- Job	
A parallel computation consisting of multiple tasks that gets spawned in response to a Spark action (e.g. save, collect); you'll see this term used in the driver's logs.

- Stage	
Each job gets divided into smaller sets of tasks called stages that depend on each other (similar to the map and reduce stages in MapReduce); you'll see this term used in the driver's logs.


每个application独立进程，便于隔离。
application之间共享，只能通过外部存储。

## 提交应用
http://spark.apache.org/docs/latest/submitting-applications.html
./bin/spark-submit

```shell
# Run application locally on 8 cores
./bin/spark-submit \
  --class org.apache.spark.examples.SparkPi \
  --master local[8] \
  /path/to/examples.jar \
  100

# Run on a Spark standalone cluster in client deploy mode
./bin/spark-submit \
  --class org.apache.spark.examples.SparkPi \
  --master spark://207.184.161.138:7077 \
  --executor-memory 20G \
  --total-executor-cores 100 \
  /path/to/examples.jar \
  1000

# Run on a Spark standalone cluster in cluster deploy mode with supervise
./bin/spark-submit \
  --class org.apache.spark.examples.SparkPi \
  --master spark://207.184.161.138:7077 \
  --deploy-mode cluster \
  --supervise \
  --executor-memory 20G \
  --total-executor-cores 100 \
  /path/to/examples.jar \
  1000

# Run on a YARN cluster
export HADOOP_CONF_DIR=XXX
./bin/spark-submit \
  --class org.apache.spark.examples.SparkPi \
  --master yarn \
  --deploy-mode cluster \  # can be client for client mode
  --executor-memory 20G \
  --num-executors 50 \
  /path/to/examples.jar \
  1000

# Run a Python application on a Spark standalone cluster
./bin/spark-submit \
  --master spark://207.184.161.138:7077 \
  examples/src/main/python/pi.py \
  1000

# Run on a Mesos cluster in cluster deploy mode with supervise
./bin/spark-submit \
  --class org.apache.spark.examples.SparkPi \
  --master mesos://207.184.161.138:7077 \
  --deploy-mode cluster \
  --supervise \
  --executor-memory 20G \
  --total-executor-cores 100 \
  http://path/to/examples.jar \
  1000

# Run on a Kubernetes cluster in cluster deploy mode
./bin/spark-submit \
  --class org.apache.spark.examples.SparkPi \
  --master k8s://xx.yy.zz.ww:443 \
  --deploy-mode cluster \
  --executor-memory 20G \
  --num-executors 50 \
  http://path/to/examples.jar \
  1000
```


## 监控
http://spark.apache.org/docs/latest/monitoring.html
http://<driver-node>:4040

## 作业调度
http://spark.apache.org/docs/latest/job-scheduling.html


## 单独集群部署模式
http://spark.apache.org/docs/latest/spark-standalone.html



## 在YARN集群上跑spark

## 在K8s上跑spark
http://spark.apache.org/docs/latest/running-on-kubernetes.html

### 安全
#### 用户身份
Dockerfile 中 USER 指令

为了能够运行spark可执行程序，UID应该包含root group作为附加组。
用户自行构建的docker镜像可以用 docker-image-tool.sh脚本 -u <uid> 来指定UID参数。

另外，Pod模板技术也可以在spark提交后台执行的时候，runAsUser 用于添加安全上下文到pod中.这个可以覆盖镜像的USER指令。但是你要注意，这个方案需要你的用户协作才行，如果是共享环境就不适合这么干。

#### volume 挂载


### 前期准备
- 一个可执行的2.3或以上的spark发行版本
- 一个1.6版本以上的k8s集群，配置kubectl.如果你没有一个正在工作的k8s集群，你也可以在本机用minikube启动一个测试集群。
  - 推荐使用最新版本minikube，并启用DNS扩展。
  - 注意：默认的minikube配置不足以运行spark应用，推荐你至少用3CPU 4G内存。
- 权限配置：你得有权限对pod进行list, create, edit, delete等操作.
你可以用命令验证一下：kubectl auth can-i <list|create|edit|delete> pods.
- 你必须配置好k8s集群的DNS.

### 运行原理
spark-submit 可以将spark应用直接提交给k8s集群，提交机制如下：
- spark会在k8s pod上创建一个spark driver；
- 这个driver会创建executor（也是跑在k8s pod上），并连接他们，并执行应用代码；
- 当应用完成时，executor pods会停止，然后被清理掉，但是driver pod会持久化日志并在k8s 中保持"completed"状态，直到最终被垃圾回收或者被手工清理掉。

注意，在completed状态时，driver pod并不占用计算和内存资源。

dirver和executor pod的调度，都是k8s来处理的。
连接k8s API是通过fabric8做的.
你也可以通过配置并使用node selector来调度driver和executor pod 只运行在可用节点的一个子集当中。


### 提交应用到k8s

#### docker镜像
spark安装目录：
现成的dockerfile: kubernetes/dockerfiles/ 
构建发布docker镜像：bin/docker-image-tool.sh
```shell
$ ./bin/docker-image-tool.sh -r <repo> -t my-tag build
$ ./bin/docker-image-tool.sh -r <repo> -t my-tag push
```

默认情况下，这个docker-image-tool.sh 构建的是基于JVM的镜像，如果你要用其他语言，可自行设置：
```Scala
# To build additional PySpark docker image
$ ./bin/docker-image-tool.sh -r <repo> -t my-tag -p ./kubernetes/dockerfiles/spark/bindings/python/Dockerfile build

# To build additional SparkR docker image
$ ./bin/docker-image-tool.sh -r <repo> -t my-tag -R ./kubernetes/dockerfiles/spark/bindings/R/Dockerfile build
```

#### 集群模式

提交示例应用Spark Pi到集群：

```shell
$ ./bin/spark-submit \
  --master k8s://https://<k8s-apiserver-host>:<k8s-apiserver-port> \
  --deploy-mode cluster \
  --name spark-pi \
  --class org.apache.spark.example.SparkPi \
  --conf spark.executor.instance=5 \
  --conf spark.kubernetes.container.image=<spark-image> \
  local:///path/to/examples.jar 
```







