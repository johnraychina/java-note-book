
# 准备知识

路由器与子网
子网：https://en.wikipedia.org/wiki/Subnetwork
子网之间过路由器做数据交换，路由器则作为子网之间的物理和逻辑边界。
子网通常使用CIDR 表示法。
举个例子：198.51.100.0/24，前面24个bit作为路由前缀，后面8位作为内部唯一编号。
ARP协议：探测子网内的ip和mac信息

IPV6： https://en.wikipedia.org/wiki/IPv6



# VPC 网络
https://cloud.google.com/vpc/docs/vpc


- 子网设置
- 路由设置
- 防火墙设置
- 通信和访问权限设置

VPC 网络使用 Linux 的 VIRTIO 网络模块来模拟以太网卡和路由器功能，但较高层次的网络堆栈（例如 ARP 查询）则使用标准网络软件进行处理。


# Cloud Interconnect 互联
https://cloud.google.com/network-connectivity/docs/interconnect/concepts/partner-overview

# VPN 连接


# 防火墙


