
Build Once, Run Anywhere

https://yeasy.gitbook.io/docker_practice/introduction/what

## 什么是docker
docker 使用Go语言开发实现，基于linux内核cgroup, namespace, OlverlayFS类的Union FS等技术,  对进程进行封装隔离，属于操作系统层面的虚拟化技术。

docker在容器的基础上，做了进一步封装，从文件系统、网络到进程隔离等，极大简化了容器的创建和维护。使得docker技术比虚拟机技术更为轻便、快捷。


## 为什么要用docker
高效利用系统资源
更快速的启动时间
一直的运行时环境
持续交付和部署
更轻松的维护和扩展



## 桌面版安装使用 Docker Desktop
https://docs.docker.com/docker-for-mac/kubernetes/



## docker 命令
最常用命令：
1. docker build 根据dockerfile构建image
2. docker run 从image启动一个container，最重要的启动参数命令行已经用-x缩写标出来了.


docker --help 可以看到命令做了归类，管理不同的对象（config, container, image, network, service, volume等）
docker image --help 又可以看到对镜像的操作，非常简明直观。

  app*        Docker App (Docker Inc., v0.9.1-beta3)
  builder     Manage builds
  buildx*     Build with BuildKit (Docker Inc., v0.5.1-docker)
  checkpoint  Manage checkpoints
  config      Manage Docker configs
  container   Manage containers
  context     Manage contexts
  image       Manage images
  manifest    Manage Docker image manifests and manifest lists
  network     Manage networks
  node        Manage Swarm nodes
  plugin      Manage plugins
  scan*       Docker Scan (Docker Inc., v0.5.0)
  secret      Manage Docker secrets
  service     Manage services
  stack       Manage Docker stacks
  swarm       Manage Swarm
  system      Manage Docker
  trust       Manage trust on Docker images
  volume      Manage volumes


### 构建 docker build 

Dockerfile:
- FROM 基于基础镜像构建
- WORKDIR 工作目录
- COPY 拷贝
- RUN 运行
- ENV 环境
- ENTRYPOINT 入口

### 执行 docker run 
1. Bind volumes: -v hostPath:containerPath

2. Bind ports:  -p hostPort:containerPort
127.0.0.1 仅本机能访问
0.0.0.1 外部可访问

3. Environment variable: --env PGUSER=pg-admin

4. Build-time arguments: --build-arg KEY=VALUE

### 上传



### 多容器app












