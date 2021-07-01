
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

## docker 概念
docker deamon
docker repository
docker image 镜像
docker container 在镜像基础上加了一层运行时可读写的 **容器存储层**.
docker 文件系统挂载
docker 网络映射

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

docker build [options] path|url

path|url: 
构建上下文，docker会先将上下文打包传给docker deamon服务端，以便COPY命令可以从服务端本地中拷贝文件到镜像中。
所以上下文应尽量精简，不要放无关的东西，你也可以用.dockerignore来剔除不需要传给docker引擎的文件。

options:
-f: dockerfile name 文件名默认是当前目录下Dockerfile，你可以用这个选项修改文件名。
-o: ouput desitination   
-q: quiet mode   
-t: tag   


Dockerfile:
- FROM 基于基础镜像构建, 如果不需要基础镜像，可以 FROM scratch 表示从空开始构建
- WORKDIR 工作目录
- COPY 拷贝
- CMD 容器启动命令
- RUN 运行 
- ENV 环境
- ENTRYPOINT 入口

#### COPY 复制文件
COPY [--chown=<user>:<group>] <源路径>... <目标路径>   
<源路径> 可以是多个，甚至可以是通配符，通配符规则满足Go的 filepath.Match 规则。   
注意：如果源路径为文件夹，复制的时候不是直接复制文件夹，而是将其中的内容复制到目标路径。   
<目标路径> 可以是容器内的绝对路径，也可以是基于工作目录（WORKDIR）的相对路径，系统会自动创建缺失目
录。   
权限：读写执行权限、文件变更时间等，都会保留，你可以加 --chown=<user>:<group> 修改所属用户和所属组。   

ADD 在COPY之上加了一些功能，而且会使镜像构建缓存失效，从而令构建变得缓慢。
在《Dockerfile最佳实践文档》中，建议尽可能用COPY，因为它语义更加明确。   

#### CMD 容器启动命令
CMD 指令格式与 RUN 相似，也是两种格式：
- shell 格式： CMD <命令>
- exec 格式： CMD ["可执行文件", "参数1", "参数2"...]
- 参数列表格式： CMD ["参数1", "参数2"...]， 在指定了 ENTRYPOINT之 指令后，用CMD指定具体参数。

注意：docker容器不是虚拟机，所以容器中的应用都应该以前台方式执行，
而不是相虚拟机、物理机那样，用systemd启动后台服务，容器内没有后台服务概念。

所以如果你这么写，容器执行后就立即退出了：
```shell
CMD service nginx start
```

#### ENTRYPOINT 入口点
ENTRYPOINT 的格式和 RUN 指令格式一样，分为 exec 和 shell 格式。
ENTRYPOINT 的目的和 CMD 一样，都是指定容器启动程序和参数。
ENTRYPOINT 在运行时也可以替代，不过比CMD 略微繁琐，
需要通过 docker run 的参数 --entrypoint 来指定。

注意：当指定了ENTRYPOINT后，CMD的含义就发生了变化，不再是直接运行命令，而是将CMD的内容作为参数传给ENTRYPOINT指定令：
```
<ENTRYPOINT> "<CMD>"
```

所以从 ENTRYPOINT 可以指定一个shell脚本，然后将 CMD 指定的参数传给脚本：
```
FROM alpine:latest
RUN addgroup -S redis && ad user -S -G redis redis
ENTRYPOINT ["docker-entrypoint.sh"]
EXPOSE 6379
CMD ["redis-server"]
```

#### ENV 设置环境变量
两种格式：
- ENV <key> <value>
- ENV <key1>=<value1> <key2>=<value2>

#### ARG 构建参数
ARG <参数名>=[<默认值>]

灵活使用ARG指令，可以在不修改Dockerfile的情况下，构建出不同镜像。   
注意：ARG指令有生效范围，如果在FROM前指定，那么只能用于FROM指定令中，
对于多阶段构建尤其要注意：在FROM之后的ARG， 必须在每个阶段分别指定。


#### VOLUME 数据卷
- VOLUME ["路径1", "路径2"]
- VOLUME 路径

这里的 /data 目录就会在容器运行时自动挂载为匿名卷，任何向 /data 写入的信息都不会记录进容器存储层，从而保证容器存储层的无状态化。
```
VOLUME /data
```
当然，运行容器时可以覆盖这个挂载设置：
```
docker run -d -v mydata:/data xxx
```
在这行命令中，就使用了mydata这个命名卷挂载到了 /data 这个位置， 替代了Dockerfile中定义的匿名卷挂载配置。

#### EXPOSE 暴露端口
EXPOSE <port1> [<port2>...]

EXPOSE 指令是申明容器运行时提供服务的端口，这只是一个声明，在容器运行时并不会因为这个声明就会开启这个端口的服务。
在Dockerfile中写入这样的声明有两个好处：
1、帮助镜像使用者理解这个镜像服务的守护端口，以方便配置映射；
2、在运行时使用随机端口映射，也就是 docker run -P 时，会自动随机映射 EXPOSE 端口。

真正开启端口：
- 手工指定 docker run -p <hostPort>:<containerPort> xxx
- 随机映射 docker run -P xxx

#### WORKDIR 指定工作目录
WORKDIR <工作路径>
注意1：cd 不能改变工作目录，要用WORKDIR改变以后各层工作目录。
注意：如果你的 WORKDIR 使用相对路径

#### USER 指定当前用户
USER <username>:[<group>]

USER指定和WORKDIR指令类似，都是改变环境状态并影响以后的层。WORKDIR 是改变工作目录，USER 是改变之后的层执行 RUN, CMD, 以及 ENTRYPOINT 这类命令的身份。

注意：USER 只是帮助你切换到指定用户而已，这个用户必须事先建立好，否则无法切换。

注意：如果以 root 执行的脚本，在执行期间希望改变身份，不要使用 su 或者 sudo ， 这些都需要比较麻烦的配置，
而且在TTY 缺失的环境下经常出错。建议使用gosu。

```sh
# 建立redis账户，并使用gosu切换到此用户启动redis server
RUN groupadd -r redis && useradd -r -g redis redis

# 下载gosu
RUN wget -O /usr/local/bin/gosu \
    "https://github.com/tianon/gosu/releases/download/1.12/gosu-amd64" \
    && chmod +x /usr/local/bin/gosu \
    && gosu nobody true

# 设置CMD，并切换到redis身份执行
CMD ["exec", "gosu", "redis", "redis-server"]    
```

#### HEALTHCHECK 健康检查
HEALTHCHECK [选项] CMD <命令> ：设置检查容器健康状况的命令
HEALTHCHECK NONE ： 如果基础镜像有健康检查命令，使用这行可以屏蔽掉其健康检查指令。

选项：
- --interval=<间隔> 两次健康检查时间间隔，默认30秒；
- --timeout=<时长> 健康检查命令运行超时时间，如果超过这个时间，本次健康检查视为失败，默认30秒；
- --retries=<次数> 当连续失败达到次数后，则将容器状态视为 unhealthy, 默认3次。

和CMD, ENTRYPOINT 一样，HEALTHCHECK 只能出现一次，如果写了多个，只有最后一个生效。

命令的返回值决定了该次健康检查的成功与否： 0-成功，1-失败，2-保留 （不要使用这个值）

示例
```sh
FROM nginx
RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*
HEALTHCHECK --interval=5s --timeout=3s \
  CMD curl -fs http://localhost/ || exit 1
```

#### ONBUILD 为他人作嫁衣裳

假设你有多个项目构建过程相似，想放到一个共享的Dockerfile，作为基础镜像：
```sh
FROM node:slim
RUN mkdir /app
WORKDIR /app
COPY ./package.json /app
RUN ["npm", "install"]
COPY . /app/
CMD ["npm", "start"]
```

但问题是，如果个别项目npm install需要加参数，怎么办？
又比如，package.json应该是用项目自己的文件，怎么办？

ONBUILD 可以解决这些问题。
1. 让我们用 ONBUILD 重新写一下基础镜像的Dockerfile:
```sh
FROM node:slim
RUN mkdir /app
ONBUILD COPY ./package.json /app
ONBUILD RUN ["npm", "install"]
ONBUILD COPY . /app/
CMD ["npm", "start"]
```

2. 然后构建基础镜像：```docker build -t my-node .``` 
这样构建基础镜像时，ONBUILD这三行不会执行。

3. 然后将项目的Dockerfile就简化为了：
```sh
FROM my-node
```

#### LABEL 为镜像添加元数据
LABEL <key>=<value>  <key>=<value> ...

### SHELL 指令
SHELL ["executable", "parameters"]

SHELL 指令可以指定 RUN，ENTRYPOINT，CMD 指令的shell，Linux中默认为 ["/bin/sh", "-c"]

```sh
SHELL ["/bin/sh", "-cex"]

# /bin/sh -cex "nginx"
ENTRYPOINT nginx
```

### Dockerfile 最佳实践

- Understand build context: 理解、精简构建上下文
- Pipe Dockerfile through stdin 利用stdin 从管道导入 Dockerfile
- Use multi-stage builds: 利用多级构建，最底层的放最前，复用构建层，缩减镜像尺寸。
- Don't install unecessary packages
- Decouple applications
- Minimize the number of layers:  RUN, COPY, ADD 才会创建层，其他指令只会创建临时中间镜像，并不会加大构建尺寸。
- Sort multi-line arguments: 多行参数，按字典排序，易维护
- Leverage build cache
- Dockerfile 优化
  - FROM
  - LABEL
  - RUN 用反斜杠将多行连成一行, apt-get update 应该和 apt-get install 一行，否则由于docker按指令缓存，可能不会构建更新的层，导致取到过期版本。当然，显示指定版本是比较保险的做法。
  - CMD 
  - EXPOSE
  - ENV
  - COPY 优于 ADD
  - ENTRYPOINT


### multi-arch 构建多种系统架构支持的Docker镜像

image manifest:  当用户拉取镜像时，docker引擎首先会查找该镜像是否有manifest列表，
如果有的话，会按docker运行环境（系统架构）找到对应镜像

查看manifest：
```sh
docker manifest inspect golang:alpine
```

0. 准备工作：   
先在Linux x86_64构建镜像 username/x8664-test   
并在Linux arm64v8构建镜像 username/arm64v8-test   
构建完成后，推送到Docker Hub   

1. 创建manifest
```sh
docker manifest create username/test \
  username/x8664-test \
  username/arm64v8-test
```

当修改一个 manifest 列表时，可以加入 -a 活这 --amend 参数。

2. 设置manifest列表
```sh
docker manifest annotate username/test \
  username/x8664-test \
  --os linux --arch x86_64

docker manifest annotate username/test \
  username/x8664-test \
  --os linux --arch arm64 --variant v8
```

这样就配置好了manifest列表。

3. 推送manifest列表到Docker Hub：
```sh
docker manifest push username/test
```


### 上传

### 执行 docker run 
1. Bind volumes: -v hostPath:containerPath

2. Bind ports:  -p ip:hostPort:containerPort | ip::containerPort| hostPort:containerPort|
ip=127.0.0.1 仅本机能访问
ip=0.0.0.1 外部可访问

3. Environment variable: --env PGUSER=pg-admin

4. Build-time arguments: --build-arg KEY=VALUE

### 使用网络

#### 容器端口映射
-P 容器内部开放的端口 随机映射到 一个外部端口
-p 容器内部开放的端口 按指定规则映射到 外部端口

#### 容器互联
```sh
# 创建网络
docker network create -d bridge my-net

# 连接容器
# 打开一号shell
docker run -it --rm --name busybox1 --network my-net busybox sh
# 打开二号shell
docker run -it --rm --name busybox2 --network my-net busybox sh
# 打开三号shell
docker container ls

# 在busybox1容器输入命令
ping busybox2
```

#### 配置DNS

如何自定义配置容器的主机名和DNS呢？
秘诀就是Docker利用虚拟文件来挂载容器3个相关配置文件。

在容器中使用mount命令可以看到挂载信息：
```sh
mount
... ...
/dev/vda1 on /etc/resolv.conf type ext4 (rw,relatime)
/dev/vda1 on /etc/hostname type ext4 (rw,relatime)
/dev/vda1 on /etc/hosts type ext4 (rw,relatime)
... ...
```

这种机制可以让宿主主机DNS信息发生更新后，所有Docker容器的DNS配置通过 
/etc/resolv.conf文件立刻得到更新

配置全部容器的DNS，也可以在 /etc/docker/deamon.json 文件中增加以下内容来设置。
```json
{
  "dns": [
    "114.114.114.114",
    "8.8.8.8"
  ]
}
```

这样每次启动的容器DNS自动配置为 ["114.114.114.114", "8.8.8.8"]。
使用以下命令来确认已经生效：
```sh
docker run -it --rm ubuntu:latest cat /etc/resolv.conf
```

docker run 参数：
- --dns=IP_ADDRESS 添加DNS服务器到容器的 /etc/resolv.conf 中
- --dns-search=DOMAIN 设定容器搜索域，当设定为.example.com时，在搜索一个名为host的主机时，DNS不仅搜索host，还会搜索host.example.com。

### 高级网络设置
当docker启动时，会自动在主机上创建一个docker0 虚拟网桥，实际上是Linux的一个bridge，可以理解为软件交换机。
它会挂载到它的网口之间进行转发。

同时，Docker随机分配一个本地未被占用的私有网段中的一个地址给docker0接口。

当创建docker容器时，同时会创建一对 veth pair 接口 （当数据包发送到一个接口时，另外一个接口也收到相同数据）。
这对接口一端在容器内，即 eth0； 
另一端在本地被挂载到docker0网桥，名称以veth开头。
通过这种方式，主机可以和容器通信，容器之间也可以互相通信。
Docker就创建了在主机和所有容器之间的一个虚拟共享网络。

如下图所示：
![网络](docker-network.png)


#### 网络访问控制原理
容器的访问控制主要通过 Linux上的 iptables 来管理和实现的。
通过命令选项，操作iptables。

容器启动时，选项 -p 或 -P 来将内部端口 映射到主机外部端口，是通过iptables的NAT实现的。


### 数据管理

volume 数据卷是一个可供一个或多个容器使用的特殊目录，它绕过UFS，可以提供很多有用的特性：
- 数据卷 可以在容器间共享使用
- 对数据卷的修改会立马生效
- 对数据卷的更新，不会影响镜像
- 数据卷默认会一直存在，即使容器被删除

注意：数据卷的使用，类似于Linux下对目录或文件进行mount，镜像中的被指定为挂载点的目录，其中的文件会被复制到数据卷中（仅数据卷为空时会复制）。

```sh
# 创建 volume
docker volume create my-vol
# 查看 volume列表
docker volume ls
# 查看volume详情
docker volume inspect my-vol
```

挂载数据卷
```sh
# 在用docker run 启动容器时，使用 --mount 选项将数据卷挂载到容器中。
# docker run 中可以挂载多个数据卷。
docker run -d -P \
  -name web \
  --mount source=my-vol, target=/usr/share/nginx/html \
  nginx:alpine
```

挂载主机目录：
```sh
docker run -d -P \
  -name web \
  --mount type=bind, source=/src/webapp, target=/usr/share/nginx/html \
  nginx:alpine
```

### 进入容器

#### attach 命令
```sh
docker run -dit ubuntu
docker container ls
docker attach <containerId>
```
注意： 如果从这个stdin中exit，会导致容器停止。


#### exec 命令
```sh
docker exec -it <containerId>
```

-i interactive
-t terminal

### 导入和导出
```sh
# 导出
docker export <containerId> ubuntu.tar
# 导入
cat ubuntu.tar | docker import - test/ubuntu:v1.0
```

## docker compose 多容器app

Compose 项目负责实现对Docker容器集群的快速编排。
从功能上看，和OpenStack中的Heat十分类似。

当我们需要多个容器配合来完成某个任务时，例如实现一个web项目，
有很多子系统，中间件，数据库等，就需要一组容器。

Compose允许用户通过一个单独的docker-compose.yml模板文件来定义
一组关联的容器作为一个项目(project).

两个重要概念：
- 服务(service): 一个应用容器，实际上可以包含若干个运行相同镜像的容器实例。
- 项目(project)：由一组关联的应用容器组成的一个完整业务单元，在docker-compose.yml文件中定义。

Compose 的默认管理对象是项目，通过子命令对项目中的一组容器进行便捷的生命周期管理。

### docker compose 命令说明
Compose V2 用Go重写了一遍，并内置作为docker cli的自命令。
注意：对于Compose来说，大部分命令的对象既可以是项目本身，也可以是项目中的服务或者容器，如果没有特别说明，命令对象将是项目，这意味着项目中所有的服务都会收到命令的影响。

```sh
docker compose --help

docker compose [OPTIONS] COMMAND

Options:
      --ansi string                Control when to print ANSI control characters ("never"|"always"|"auto") (default "auto")
      --env-file string            Specify an alternate environment file.
  -f, --file stringArray           Compose configuration files
      --profile stringArray        Specify a profile to enable
      --project-directory string   Specify an alternate working directory
                                   (default: the path of the Compose file)
  -p, --project-name string        Project name

Commands:
  build       Build or rebuild services
  convert     Converts the compose file to platform's canonical format
  create      Creates containers for a service.
  down        Stop and remove containers, networks
  events      Receive real time events from containers.
  exec        Execute a command in a running container.
  images      List images used by the created containers
  kill        Force stop service containers.
  logs        View output from containers
  ls          List running compose projects
  pause       pause services
  port        Print the public port for a port binding.
  ps          List containers
  pull        Pull service images
  push        Push service images
  restart     Restart containers
  rm          Removes stopped service containers
  run         Run a one-off command on a service.
  start       Start services
  stop        Stop services
  top         Display the running processes
  unpause     unpause services
  up          Create and start containers

```

### compose模板文件
compose模板文件默认名称是docker-compose.yml，它是YAML格式的。
其中的指令与dockerfile 异曲同工。

- image 指向一个镜像，并配置网络、数据卷等。
```yml
version: "3"

services:
  webapp:
    image: examples/web
    ports:
      - "80:80"
    volumes:
      - "/data"
```


- build 指定构建镜像需要的Dockerfile，上下文，参数等。
```yml
version: "3"

services:
  webapp:
    build: ./dir
```

- cap_add, cap_drop 指定容器的内核能力（capacity）分配
- command 覆盖容器启动后默认执行的命令
- configs 仅用于swarm mode
- cgroup_parent 指定cgroup组，意味着将继承该组的限制。
- container_name 指定容器名称，默认情况下，会使用 project_service_seq 格式.
- deploy 仅用于swarm mode
- devices 指定设备映射关系
- depends_on 解决容器依赖、启动先后问题。
- dns 自定义DNS服务器，可以是一个值或者列表。
- dns_search 配置DNS搜索域，可以是一个值或者列表。
- tmpfs 挂载一个tmpfs文件系统到容器
- env_files 从文件中获取环境变量，可以是一个或者列表。如果docker compose -f 指定compose模板文件，则env_file中的路径会基于模板文件路径。如果变量名与environment指令冲突，则按惯例，以后者为准。
- environment 设置环境变量。你可以使用数组或者字典两种格式。
- expose 暴露端口，但不映射到宿主机，只能被连接的服务访问。仅指定内部端口为参数。
- external_links 不建议使用
- extra_hosts 类似docker中的 --add-host参数，指定额外的host名称映射信息。
- healthcheck 健康检查配置
- image 镜像配置
- labels 添加元数据信息
- logging 日志配置
- network_mode 设置网络模式
- networks 配置容器连接的网络
- pid 与主机系统共享进程命名空间，打开该选项的容器之间，以及容器和宿主机系统之间可以通过进程ID来相互访问和操作。
- ports 映射宿主机端口 到容器端口
- secrets 存储敏感数据
- security_opt 指定容器模板标签（label）机制的默认属性（用户，决赛，类型，级别等）。
- stop_signal 设置另一个信号来停止容器。在默认情况下使用的是SIGTERM。
- sysctls 设置容器内核参数
- ulimits 指定容器上限值，例如最大进程数，文件句柄数等。
- volumes 数据卷锁挂载路径设置。
- 其他指令基本和docker run对应参数基本一致：
entrypoint,working_dir
domainname, hostname, ipc, mac_address, privileged, read_only, shm_size, restart, stdin_open, tty, user.

#### 读取环境变量
compose 模板文件支持读取 主机系统环境变量、当前目录下的 .evn 文件中的变量.

例如
```yml
version: "3"
services:
  db:
    image: "mongodb:${MONGO_VERSION}"
```

如果执行 MONGO_VERSION=3.2, docker compose up 则会启动一个mongo:3.2镜像的容器。
如果在当前目录创建 .env 并写MONGO_VERSION，则执行docker compose 命令时会从该文件读取变量。
```
echo "MONGO_VERSION=3.6" > .env
```


## Swarm mode 集群管理和编排

### 基本概念
Swarm 是使用 Swarmkit 构建的Docker引擎内置（原生）的集群管理和编排工具。

使用Swarm 集群之前需要了解几个概念：
- 节点 Node
- 服务和任务
### 节点
运行Docker的主机可以主动初始化一个Swarm集群，或者加入一个已存在的Swarm集群，这个运行Docker的主机就成为一个Swarm集群的节点（Node）。

节点分为 管理节点（manager）和 工作节点（worker）。

管理节点用于Swarm集群的管理，docker swarm 命令基本只能在管理节点执行（节点退出可以在工作节点上执行）。
一个Swarm集群可以有多个管理节点，但只有一个管理节点可以成为leader，leader通过raft协议实现。

工作节点是任务执行节点，管理节点将服务（service）下发至工作节点执行。
管理节点默认也作为工作节点。你可以配置让服务只运行在管理节点。   

来自docker官网的这张图片形象地展示了集群中管理节点和工作节点的关系。
![部署图](swarm-diagram.png)

### 服务和任务

任务（Task）是Swarm中最小的调度单位，目前来说就是一个单一容器。

服务（Services）是指一组任务的集合，服务定义了任务的属性。

服务有两种模式：
- replicated services 按照一定规则在各个工作节点上运行指定个数的任务
- global services  每个工作节点上运行一个任务

两种模式通过 docker service create 的 --mode 参数指定。

来自Docker官网的这张图形象展示了容器、任务、服务之间的关系。
![服务](services-diagram.png)

### docker app
### docker secret
### docker node: Manage Swarm nodes
