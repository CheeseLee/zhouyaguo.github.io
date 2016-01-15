---
layout: post
title: "[openstack] [neutron] 网络基础"
comments: true
categories:
- openstack
---

Ethernet
--------

- OSI第二层
- NIC=network interface card， 网卡
- mac地址（48bits）
- frame
- 对应的名词： local network, layer 2, L2, link layer, data link layer
- 查看mac地址： ip link show eth0
- 二层设备switch（交换机）
- Ethernet支持broadcasts（向ff:ff:ff:ff:ff:ff发包），例如ARP，DHCP
- Ethernet network因为支持broadcasts，所以又被称为broadcast domain（广播域）
- NIC可设置为promiscuous mode（混杂模式）：网卡将所有frames都传给OS，就算不是发给自己的
- switch（交换机）内部会学习mac与port的对应关系table，称为forwarding table or forwarding information base (FIB)

vlan
----

作用：把一个交换机变成多个虚拟的交换机，vlan用来隔离broadcast domain（广播域）

- 不同vlan之间的机器互相看不到
- 当一个交换机的端口被设置到某个vlan中，则称这个端口为 _access port_
- 当一个交换机的端口被设置可以让所有vlan通行，并且打上vlan tag时，则称这个端口为 _trunk port_
  - 在openstack环境中，如果让物理交换机使用vlan的方式来隔离租户的话，必须将所有的端口都设置为 _trunk port_


子网
----

例如ip为192.168.1.5，掩码是255.255.255.0

  - 则`CIDR`的写法就是192.168.1.5/24，CIDR同时描述了ip和掩码
  - 子网的描述方法就是192.168.1.0

ARP
---

arping命令可以发起arp报文

交互过程：

1. To: everybody (ff:ff:ff:ff:ff:ff). I am looking for the computer who has IP address 192.168.1.7. Signed: MAC address fc:99:47:49:d4:a0.
2. Host B responds with a response like this: To: fc:99:47:49:d4:a0. I have IP address 192.168.1.7. Signed: MAC address 54:78:1a:86:00:a5.

DHCP
----

DHCP使用UDP报文。

交互过程：

1. The client sends a discover (“I’m a client at MAC address 08:00:27:b9:88:74, I need an IP address”)
1. The server sends an offer (“OK 08:00:27:b9:88:74, I’m offering IP address 10.10.0.112”)
1. The client sends a request (“Server 10.10.0.131, I would like to have IP 10.10.0.112”)
1. The server sends an acknowledgement (“OK 08:00:27:b9:88:74, IP 10.10.0.112 is yours”)

openstack使用dnsmasq作为dhcp server，可以在syslog中看到dnsmasq的日志。

router
-------

- router实现不同local network的机器间的通信
- router连接了不同的local network
- router有多个ip地址，每个local network都有一个。

ip
---

- 对应的名词： layer 3, L3, and network layer
- 一个host要发ip包时，先查看路由表，才知道next hop是谁

查看路由表的三种方法：

- ip route show
- route -n
- netstat -rn

`ip route get 10.0.2.14` 的输出是： destination为10.0.2.14的路由

DHCP、DNS、NTP、VXLAN都基于UDP

UDP支持一对多通信，UDP通过IP multicast来将udp包发送到多个已加入group的receivers。VXLAN就使用了IP multicast

ping和`mtr`命令使用了ICMP，其中`mtr`结合了ping和traceroute的功能。

tunnel技术
---------

tunnel技术就是将一种protocol的报文作为payload嵌入另一种protocol中传输

  - GRE： 把private ip address的ip数据包用公网ip地址包装后传输
  - VXLAN： vxlan把L2的 Ethernet frames 包装成UDP，把不同网络的虚拟机组合在一起，组成一个logic network。


namespace
---------

linux提供了network和process的namespace，用于隔离进程和网络。如果进程运行在namespace中，那么它只能看到该namespace中的进程。

- 每个网卡（eth0）都属于某个namespace，如果不显式地指定，则eth0属于默认namespace。
- 不同的network namespace有不同的路由表，当你想让一个destination ip在不同场景下走不同的路由时，namespace就发挥了作用。
- 不同的network namespace有不同的iptables
- namespace相关的命令是 `ip netns exec namespace名字 COMMAND`

Virtual routing and forwarding (VRF)
  - VRF意思就是一个router内部可以有多个路由表，指的就是network namespace所能达到的效果

NAT
---

- NAT含义是Network Address Translation，就是修改ip报文的头部的source ip或者destination ip，NAT通常都是router的行为。在openstack中，router的角色由iptables扮演。
- NAT通常由router执行。在linux中，iptables可以作为NAT执行的角色。


`SNAT`：修改源ip，用于private address访问外网的server，SNAT是为了解决ip地址不够用的情况。

RFC 1918保留了3个网段作为private addresses：

- 10.0.0.0/8 (即 10.0.0.0 - 10.255.255.255)
- 172.16.0.0/12 (即 172.16.0.0 - 172.31.255.255)
- 192.168.0.0/16 (即 192.168.0.0 - 192.168.255.255)

openstack中，sender和receiver之间有个NAT router，router既修改ip，又修改port，同时会记录修改的对应关系，当外面发回报文时，
router会准确的转给后面的private address。这种同时修改port的SNAT又称为Port Address Translation (PAT)，这种做法又称为`NAT overload`

`DNAT`：修改目的ip

openstack中用DNAT将来自instance的报文路由到metadata service去。(比如ssh keys就要通过metadata service获取)

当instance中的应用通过HTTP GET方式访问169.254.169.254时，由于这个ip不存在，于是openstack用DNAT的方式将目标ip修改为metadata service的ip

`One-to-one NAT`

NAT router维护private address和public address的一对一关系，openstack用来实现`floating ip`
