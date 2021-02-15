
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


### 构建

FROM 基于基础镜像构建
WORKDIR 工作目录
COPY 拷贝
RUN 运行
ENV 环境
ENTRYPOINT 入口

### 打标

### 上传


### 持久化和挂载

#### named volume 
1. 创建  docker volume create todo-db
2. 挂载  docker run -dp 3000:3000 -v todo-db:/etc/todos getting-started
3. 查看  docker volume inspect todo-db
[
    {
        "CreatedAt": "2019-09-26T02:18:36Z",
        "Driver": "local",
        "Labels": {},
        "Mountpoint": "/var/lib/docker/volumes/todo-db/_data",
        "Name": "todo-db",
        "Options": {},
        "Scope": "local"
    }
]
注意：While running in Docker Desktop, the Docker commands are actually running inside a small VM on your machine. If you wanted to look at the actual contents of the Mountpoint directory, you would need to first get inside of the VM.


### use bind mounts
named volume: 你控制不了数据存在哪里
bind mounts: 你可以控制挂载点
使用场景：代码动态加载、通信等

https://docs.docker.com/get-started/06_bind_mounts/
```shell
 docker run -dp 3000:3000 \
     -w /app -v "$(pwd):/app" \
     node:12-alpine \
     sh -c "yarn install && yarn run dev"
```

### 多容器app












