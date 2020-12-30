
使用场景：
- 外网**安全地访问**企业内网
 -通过多虚拟网关技术实现业务隔离

原理：
1. 报文封装：VPN Server IP + 加密封装(原始报文)
2. VPN Server转发

[GFW原理和ShadowSocks](https://www.youtube.com/watch?v=k80cu16M-rw)

[VPN是什么？(史上最好懂版本)](https://www.vpndada.com/what-is-vpn-cn/)

[VPN原理](https://yuerblog.cc/2017/01/03/how-vpn-works-and-how-to-setup-pptp/)

[SSL VPN技术原理](https://cshihong.github.io/2019/05/11/SSL-VPN%E6%8A%80%E6%9C%AF%E5%8E%9F%E7%90%86/)

SSL保护的是应用层的数据，可以针对某个应用做具体保护。而IPSec VPN针对的是整个网络层。无法做精细化控制。
所以SSL VPN出现之前，IPSec, L2TP VPN等早期出现的VPN技术虽然能实现远程接入，但是存在如下缺陷：
- 远程用户终端（比如你的笔记本）需要安装指定的客户端软件，导致**网络部署、维护比较麻烦。**
- IPSec/L2TP VPN的**配置繁琐。**
- 网络管理人员无法对远程用户访问企业内网资源的权限做**精细化的控制。**


[IPsec-VPN技术原理](https://cshihong.github.io/2019/04/03/IPSec-VPN%E5%9F%BA%E6%9C%AC%E5%8E%9F%E7%90%86/)

[VPN原理与简单应用](https://zhuanlan.zhihu.com/p/71536075)
Tunnel隧道技术
就像在错综复杂的网络中打通了一条隧道一样，把两个公司连接在一起，隧道的出口，就是两个路由器的公网ip地址（因为两边都是公网ip，所以他们是相互可达的，这是构建隧道的前提）。
隧道好比是一条直连的网线，把两个路由器连接了起来。
所以路由器还要有两个虚拟的接口（tunnel接口），然后再把这两个虚拟接口配置上相同网段的ip地址。



## 端到端VPN
这种通过两端路由器建立隧道然后实现两个私网互通的VPN形态成为**端到端VPN**:
1. 设置两端路由器，启用隧道接口，并设置隧道源和目的
interface Tunnel0
tunnel source, tunnel destination

2. 添加路由表：默认还是走公网，设置之后才会走VPN隧道.
ip route [ip] [mask] tunnel 0

## 点到端VPN(PPTP VPN)
出差带个笔记本，连接公司内网。当然前提条件还是要能ping通公司外网ip。

1. 公司路由器开启PPTP VPN的服务：
- 先建立一个地址池，这个地址池用来给笔记本隧道接口下发地址。
注意：PPTP VPN在建立的时候，远端隧道接口（笔记本）的地址必须和要访问的服务器（公司）在同一网段，否则无法访问。

- 定义一个虚拟拨号模块，该模块借用物理接口的IP地址，指定对端IP地址获得方式为从本地地址池获得，定义认证类型 pap chap ms-chap,然后开启远端拨入功能，拨入协议为pptp，并与虚拟模块1相关联。

- 最终定义一个用于远端拨入的用户名和密码。

2. 笔记本使用【虚拟专用网络连接】向导，建立与公司VPN连接
注意：此外，还要做一下隧道分离，如果不做，当笔记本连接到VPN后，将不能上网，因为所有数据都走了隧道，隧道分离方式如下：



