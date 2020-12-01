
https://zhuanlan.zhihu.com/p/83398714
https://developer.ibm.com/articles/j-zerocopy/
https://www.cnblogs.com/xiaolincoding/p/13719610.html


心理表征：

用户态：read file       write socket
------|-------------------|-----
内核态：read buffer     socket buffer
------|-------------------|-----
    磁盘文件              网卡


磁盘-->内核磁盘缓冲区-->用户态-->内核socket缓冲区-->网卡


# 基于memory map+write的零拷贝
**Java层面**

java.nio.channels.FileChannel.map

**OS层面**


# 基于sendfile的零拷贝

**Java层面**

java.nio.channels.FileChannel.transferTo

**OS层面**

1) Linux kernels 2.1引入sendfile
```c
    ssize_t sendfile(int out_fd, int in_fd, off_t *offset, size_t count);
```

2) Linux kernels 2.4修改了sendfile支持DMA gather copy
需要网卡支持gather operations



