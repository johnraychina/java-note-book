
https://draveness.me/docker/


docker容器基于[操作系统提供的三大基石](docker-arch-in-linux.png)
- Namespaces 隔离资源(pid进程、ipc进程间通信、net网络、mnt文件等)
- CGroups(Control Groups): 隔离物理资源（CPU、内存、磁盘）
- UionFS(Layer Capabilities): 镜像分层构建和容器运行 需要基于联合文件系统(aufs, overlay2, devicemapper, zfs, vfs等).

# Namespaces 隔离资源

命名空间 (namespaces) 是 Linux 为我们提供的用于分离进程树、网络接口、挂载点以及进程间通信等资源的方法。

## 进程隔离
Linux 的命名空间机制提供了以下七种不同的命名空间，包括 CLONE_NEWCGROUP、CLONE_NEWIPC、CLONE_NEWNET、CLONE_NEWNS、CLONE_NEWPID、CLONE_NEWUSER 和 CLONE_NEWUTS，通过这七个选项我们能在创建新的进程时设置新进程应该在哪些资源上与宿主机器进行隔离。


docker创建container:
所有命名空间相关的设置 spec 最后都会作为 Create 函数的入参在创建新的容器时进行设置。
所有与命名空间的相关的设置都是在上述的两个函数中完成的，Docker 通过命名空间成功完成了与宿主机进程和网络的隔离。
```Go
daemon.containerd.Create(context.Background(), container.ID, spec, createOptions)
```

```Go
containerRouter.postContainersStart
└── daemon.ContainerStart
    └── daemon.createSpec
        └── setNamespaces
            └── setNamespace
```

## 网络通信
如果 Docker 的容器通过 Linux 的命名空间完成了与宿主机进程的网络隔离，但是却又没有办法通过宿主机的网络与整个互联网相连，就会产生很多限制，所以 Docker 虽然可以通过命名空间创建一个隔离的网络环境，但是 Docker 中的服务仍然需要与外界相连才能发挥作用。


每一个使用 docker run 启动的容器其实都具有单独的网络命名空间，Docker 为我们提供了四种不同的网络模式，Host、Container、None 和 Bridge 模式。

[Bridge桥接模式](docker-network-topology.png)


### libnetwork
todo

## 挂载点

### chroot
todo

# CGroups (Control Groups)
隔离物理资源（CPU、内存、磁盘IO和网络带宽）

# Layer Capabilities
docker image: 本质上是就是一个压缩文件。

docker container: 对镜像文件上加了一个可写的层（文件系统）

docker 中的每一个镜像都是由一系列只读的层组成的，Dockerfile 中的每一个命令都会在已有的只读层上创建一个新的层：
当镜像被 docker run 命令创建时就会在镜像的最上层添加一个可写的层，也就是容器层，所有对于运行时容器的修改其实都是对这个容器读写层的修改。

docker镜像分层构建，容器之间共用。
联合文件系统: aufs、devicemapper、overlay2、zfs 和 vfs 等.
参见[Storage Driver](https://docs.docker.com/storage/storagedriver/select-storage-driver/).

## AUFS
UnionFS 其实是一种为 Linux 操作系统设计的用于把多个文件系统『联合』到同一个挂载点的文件系统服务。
而 AUFS 即 Advanced UnionFS 其实就是 UnionFS 的升级版，它能够提供更优秀的性能和效率。

AUFS 作为联合文件系统，它能够将不同文件夹中的层联合（Union）到了同一个文件夹中，这些文件夹在 AUFS 中称作分支，整个『联合』的过程被称为联合挂载（Union Mount）。


# 参考文档
- Chapter 4. Docker Fundamentals · Using Docker by Adrian Mount
- TECHNIQUES BEHIND DOCKER
- Docker overview
- Unifying filesystems with union mounts
- DOCKER 基础技术：AUFS
- RESOURCE MANAGEMENT GUIDE
- Kernel Korner - Unionfs: Bringing Filesystems Together
- Union file systems: Implementations, part I
- IMPROVING DOCKER WITH UNIKERNELS: INTRODUCING HYPERKIT, VPNKIT AND DATAKIT
- Separation Anxiety: A Tutorial for Isolating Your System with Linux Namespaces
- 理解 chroot
- Linux Init Process / PC Boot Procedure
- Docker 网络详解及 pipework 源码解读与实践
- Understand container communication
- Docker Bridge Network Driver Architecture
- Linux Firewall Tutorial: IPTables Tables, Chains, Rules Fundamentals
- Traversing of tables and chains
- Docker 网络部分执行流分析（Libnetwork 源码解读）
- Libnetwork Design
- 剖析 Docker 文件系统：Aufs与Devicemapper
- Linux - understanding the mount namespace & clone CLONE_NEWNS flag
- Docker 背后的内核知识 —— Namespace 资源隔离
- Infrastructure for container projects
- Spec · libcontainer
- DOCKER 基础技术：LINUX NAMESPACE（上）
- DOCKER 基础技术：LINUX CGROUP
- 《自己动手写Docker》书摘之三： Linux UnionFS
- Introduction to Docker
- Understand images, containers, and storage drivers
- Use the AUFS storage driver