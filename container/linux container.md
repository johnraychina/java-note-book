https://zhuanlan.zhihu.com/p/56985005


Why: 提高计算机资源使用率
关键作用：资源隔离，资源管理

# 虚拟化技术
将计算机物理资源进行抽象，转换为虚拟的计算机资源提供给程序使用的技术。
计算机资源包括：计算（CPU），存储（内存、磁盘），网络

# 虚拟机技术
操作系统级别的隔离。
在HostOS上搭载虚拟操作系统，常见的Hypervisor，VMware Workstation、Xen，JVM、HHVM等。

# 容器技术
进程级别的隔离。
无指令转换开销，运行效率高。

# 容器发展
1979 Unix chroot
2000 FreBSD Jails
2007 Google ControlGroups
2008 CGroups+Namespace==> LXC

OCI定义了容器运行时标准
containerd 容器技术标准化之后的产物，负责镜像管理（镜像、元信息）、容器执行（调用最终运行时组件）。
runC 是Docker按照开放容器格式标准制定的一种具体实现，实现了容器启停。


# 容器运行时
Runtime 是容器真正运行的地方，它需要与OS紧密协作，为容器提供运行环境.
- LXC Linux上老牌容器runtime, docker最初也是用LXC作为runtime.
- runC Docker自己开发的容器runtime, 符合OCI规范，也是目前docker默认的runtime.
- rkt CoreOS开发的容器runtime, 符合OCI规范，因而能够运行docker的容器.

