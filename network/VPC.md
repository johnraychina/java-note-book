[Google VPC](https://cloud.google.com/vpc/docs/overview)
您可以像看待物理网络一样看待 VPC 网络，区别是后者是在 Google Cloud 中进行的虚拟化。VPC 网络是一种全球化资源，它由数据中心内的一系列区域性虚拟子网组成，所有子网通过全球广域网连接。在 Google Cloud 中，VPC 网络在逻辑上彼此隔离。



[阿里云VPC](https://help.aliyun.com/learn/learningpath/vpc.html)
专有网络是您自己独有的云上私有网络。您可以完全掌控自己的专有网络，例如选择IP地址范围、配置路由表和网关等，您可以在自己定义的专有网络中使用阿里云资源如云服务器、云数据库RDS版和负载均衡等。
您可以将专有网络连接到其他专有网络或本地网络，形成一个按需定制的网络环境，实现应用的平滑迁移上云和对数据中心的扩展。

组成部分：每个VPC都由一个路由器、至少一个私网网段和至少一个交换机组成。

基于目前主流的隧道技术(Tunnel)， 虚拟专用网络


[AWS VPC如何工作](https://docs.aws.amazon.com/zh_cn/vpc/latest/userguide/how-it-works.html)
[AWS VPC和子网](https://docs.amazonaws.cn/vpc/latest/userguide/VPC_Subnets.html)
Virtual Private Cloud (VPC) 是仅适用于您的 AWS 账户的虚拟网络。它在逻辑上与 AWS 云中的其他虚拟网络隔绝。您可以在您的 VPC 内启动您的 AWS 资源，例如 Amazon EC2 实例。您可以为 VPC 指定 IP 地址范围、添加子网、关联安全组以及配置路由表。

子网是您的 VPC 内的 IP 地址范围。您可以在指定子网内启动 AWS 资源。对必须连接互联网的资源使用公有子网，而对将不会连接到互联网的资源使用私有子网。

如需保护您在每个子网中的 AWS 资源，您可以使用多安全层，包括安全组和访问控制列表 (ACL)。
您可以选择将一个 IPv6 CIDR 块与 VPC 关联，并将 IPv6 地址分配给 VPC 中的实例。