

Datagram 数据报文：计算机
    Connection-less

Virtual Circuit 虚拟电路: 电话
    Connection-ful
    call setup/teardown
    VC Identifier instead of IpAdress
    router state
    resource reservation for predictable services



Forwarding Table 路由转发表
    prefix matching 前缀匹配
    

What's inside a router? 
路由器做了什么，他的构造是什么样的？

Input ports
Output ports
Routing processor
Switching fabric


# DNS记录类型

- cname record: canonical name record 域名别名
- dname record: delegation name record 批量域名别名


## CNAME Record
https://en.wikipedia.org/wiki/CNAME_record

cname是canonical name record的简称，是一种DNS的记录。

这种记录 将一个域名映射到另外一个域名。

你可以很方便地将多个服务部署在一个ip上，通过不同端口访问（比如ftp服务器，web服务器）。

比如：
你可以将 ftp.example.com 和 www.example.com 映射到DNS记录 example.com上，

而example.com 是一条指向IP地址的A记录。

如果需要修改IP地址，那么只需要改example.com这一条记录。
## DNAME Record






