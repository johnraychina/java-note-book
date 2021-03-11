
https://zhuanlan.zhihu.com/p/83398714
https://developer.ibm.com/articles/j-zerocopy/
https://www.cnblogs.com/xiaolincoding/p/13719610.html


心理表征：

用户态：read file       write socket
------|-------------------|-----
内核态：read buffer     socket buffer
------|-------------------|-----
    磁盘文件              网卡


- 普通拷贝: read+write, 4次上下文切换，4次拷贝
- mmap + write: 4次上下文切换，3次拷贝。
- sendfile: 2次上下文切换，3次拷贝。

# 基于memory map+write的零拷贝
**Java API:**   java.nio.channels.FileChannel.map

**OS API:**   mmap(), write()

2次系统调用，4次上下文切换，3次拷贝。

# 基于sendfile的零拷贝

[sendfile零拷贝](ZeroCopy-sendfile.png)

磁盘-->内核磁盘缓冲区-->内核socket缓冲区-->网卡
1次sendfile系统调用，2次上下文切换，3次拷贝。


**Java API:** java.nio.channels.FileChannel.transferTo


**Linux API:**
1) Linux kernels 2.1引入sendfile
```c
    ssize_t sendfile(int out_fd, int in_fd, off_t *offset, size_t count);
```

2) Linux kernels 2.4修改了sendfile支持DMA gather copy
需要网卡支持gather operations




# PageCache
文件传输过程，其中第一步都是先需要先把磁盘文件数据拷贝「内核缓冲区」里，这个「内核缓冲区」实际上是磁盘高速缓存（PageCache）。
由于零拷贝使用了 PageCache 技术，可以使得零拷贝进一步提升了性能。

PageCache主要起到2个作用：1.缓存最近被访问的数据；2.预读功能；
但是，在传输大文件（GB 级别的文件）的时候，PageCache 会不起作用，那就白白浪费 DMA 多做的一次数据拷贝，造成性能的降低，即使使用了 PageCache 的零拷贝也会损失性能

这是因为如果你有很多 GB 级别文件需要传输，每当用户访问这些大文件的时候，内核就会把它们载入 PageCache 中，于是 PageCache 空间很快被这些大文件占满。

所以，针对大文件的传输，不应该使用 PageCache，也就是说不应该使用零拷贝技术，因为可能由于 PageCache 被大文件占据，而导致「热点」小文件无法利用到 PageCache，这样在高并发的环境下，会带来严重的性能问题。

# 大文件传输用什么方式实现？

当文件太大，使用「异步 I/O + 直接 I/O」，否则使用「零拷贝技术」。

例如Nginx可以配置:  directio 1024m; 
