---
layout: post
title: "[openstack] [neutron] 数据包流向：vm->外网"
comments: true
categories:
- openstack
---

通过命令查看网卡和交换机的连通性
------------------------

- 查看vm vif： `cat /opt/stack/data/nova/instances/3065f30b-00fb-4d58-9028-7b76e9e4b262/libvirt.xml`

  ```  
  <interface type="bridge">
    <mac address="fa:16:3e:fa:47:7a"/>
    <model type="virtio"/>
    <driver name="qemu"/>
    <source bridge="qbrd6acfbb9-49"/>   --对应bridge名字
    <target dev="tapd6acfbb9-49"/>      --对应bridge上的插口
  </interface>
  ```

- 查看linux bridge： `brctl show` 。bridge的两个口，一个连vm，一个连ovs

  ```
  brctl show
  bridge name	bridge id		STP enabled	interfaces
  qbrd6acfbb9-49		8000.6a99c16289a0	no		qvbd6acfbb9-49
  							                            tapd6acfbb9-49
  ```
- `ethtool -S qvbd6acfbb9-49` 找出peer

  ```
  ethtool -S qvbd6acfbb9-49
  NIC statistics:
       peer_ifindex: 41
  ```

- `ip addr` 找到peer是 qvod6acfbb9-49

  ```
  41: qvod6acfbb9-49: <BROADCAST,MULTICAST,PROMISC,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast master ovs-system state UP qlen 1000
      link/ether 92:16:c2:20:b9:61 brd ff:ff:ff:ff:ff:ff
      inet6 fe80::9016:c2ff:fe20:b961/64 scope link
         valid_lft forever preferred_lft forever
  ```

- `ovs-vsctl show`，找出qvod6acfbb9-49，属于vlan1，位于br-int

  ```
  sudo ovs-vsctl show
  353e19b5-59fe-4937-bcd3-f0c6a6cd22fa
      Bridge br-int
          fail_mode: secure
          Port "tap81c26343-a6"                    --连dhcp namespace
              tag: 1
              Interface "tap81c26343-a6"
                  type: internal
          Port int-br-ex                           --连br-ex
              Interface int-br-ex
                  type: patch
                  options: {peer=phy-br-ex}
          Port "qvod6acfbb9-49"                    --连linux bridge
              tag: 1
              Interface "qvod6acfbb9-49"
          Port "qg-6b71d14f-27"                    --连router的外网ip网卡
              tag: 2
              Interface "qg-6b71d14f-27"
                  type: internal
          Port "qr-79cbf0f1-a2"                    --连router的内网ip网卡
              tag: 1
              Interface "qr-79cbf0f1-a2"
                  type: internal
          Port br-int
              Interface br-int
                  type: internal
          Port patch-tun                           --连br-tun
              Interface patch-tun
                  type: patch
                  options: {peer=patch-int}
      Bridge br-ex
          Port phy-br-ex                           --连br-int
              Interface phy-br-ex
                  type: patch
                  options: {peer=int-br-ex}
          Port "eth0"                              --连物理网卡，通往外网
              Interface "eth0"
          Port br-ex
              Interface br-ex
                  type: internal
      Bridge br-tun
          fail_mode: secure
          Port br-tun
              Interface br-tun
                  type: internal
          Port patch-int                           --连br-int
              Interface patch-int
                  type: patch
                  options: {peer=patch-tun}
      ovs_version: "2.4.0"
  ```

- `ovs-ofctl show br-int` 查看qvod6acfbb9-49对应的编号是 7

  ```
  sudo ovs-ofctl show br-int           
  OFPT_FEATURES_REPLY (xid=0x2): dpid:0000327c6d6b274e
  n_tables:254, n_buffers:256
  capabilities: FLOW_STATS TABLE_STATS PORT_STATS QUEUE_STATS ARP_MATCH_IP
  actions: output enqueue set_vlan_vid set_vlan_pcp strip_vlan mod_dl_src mod_dl_dst mod_nw_src mod_nw_dst mod_nw_tos mod_tp_src mod_tp_dst
   1(int-br-ex): addr:86:a0:f6:fe:17:8b
       config:     0
       state:      0
       speed: 0 Mbps now, 0 Mbps max
   2(patch-tun): addr:3a:34:63:b7:ab:e5
       config:     0
       state:      0
       speed: 0 Mbps now, 0 Mbps max
   3(tap81c26343-a6): addr:00:00:00:00:00:00
       config:     PORT_DOWN
       state:      LINK_DOWN
       speed: 0 Mbps now, 0 Mbps max
   4(qr-79cbf0f1-a2): addr:00:00:00:00:00:00
       config:     PORT_DOWN
       state:      LINK_DOWN
       speed: 0 Mbps now, 0 Mbps max
   5(qg-6b71d14f-27): addr:00:00:00:00:00:00
       config:     PORT_DOWN
       state:      LINK_DOWN
       speed: 0 Mbps now, 0 Mbps max
   7(qvod6acfbb9-49): addr:92:16:c2:20:b9:61
       config:     0
       state:      0
       current:    10GB-FD COPPER
       speed: 10000 Mbps now, 0 Mbps max
   LOCAL(br-int): addr:32:7c:6d:6b:27:4e
       config:     PORT_DOWN
       state:      LINK_DOWN
       speed: 0 Mbps now, 0 Mbps max
  OFPT_GET_CONFIG_REPLY (xid=0x4): frags=normal miss_send_len=0
  ```

- `ovs-ofctl dump-flows br-int` 查看ovs的流表处理。

  ```
  sudo ovs-ofctl dump-flows br-int
  NXST_FLOW reply (xid=0x4):
   cookie=0x9c869d9a8de98c57, duration=147238.155s, table=0, n_packets=0, n_bytes=0, idle_age=65534, hard_age=65534, priority=10,icmp6,in_port=7,icmp_type=136 actions=resubmit(,24)
   cookie=0x9c869d9a8de98c57, duration=147238.133s, table=0, n_packets=50, n_bytes=2100, idle_age=901, hard_age=65534, priority=10,arp,in_port=7 actions=resubmit(,24)
   cookie=0x9c869d9a8de98c57, duration=148750.891s, table=0, n_packets=669828, n_bytes=119643190, idle_age=0, hard_age=65534, priority=3,in_port=1,vlan_tci=0x0000 actions=mod_vlan_vid:2,NORMAL
   cookie=0x9c869d9a8de98c57, duration=148781.700s, table=0, n_packets=85, n_bytes=15357, idle_age=65534, hard_age=65534, priority=2,in_port=1 actions=drop
   cookie=0x9c869d9a8de98c57, duration=148781.895s, table=0, n_packets=7722, n_bytes=852318, idle_age=40, hard_age=65534, priority=0 actions=NORMAL
   cookie=0x9c869d9a8de98c57, duration=148781.885s, table=23, n_packets=0, n_bytes=0, idle_age=65534, hard_age=65534, priority=0 actions=drop
   cookie=0x9c869d9a8de98c57, duration=147238.180s, table=24, n_packets=0, n_bytes=0, idle_age=65534, hard_age=65534, priority=2,icmp6,in_port=7,icmp_type=136,nd_target=fd61:19a:66c8:0:f816:3eff:fefa:477a actions=NORMAL
   cookie=0x9c869d9a8de98c57, duration=147238.167s, table=24, n_packets=0, n_bytes=0, idle_age=65534, hard_age=65534, priority=2,icmp6,in_port=7,icmp_type=136,nd_target=fe80::f816:3eff:fefa:477a actions=NORMAL
   cookie=0x9c869d9a8de98c57, duration=147238.143s, table=24, n_packets=50, n_bytes=2100, idle_age=901, hard_age=65534, priority=2,arp,in_port=7,arp_spa=10.0.0.3 actions=NORMAL
   cookie=0x9c869d9a8de98c57, duration=148781.875s, table=24, n_packets=0, n_bytes=0, idle_age=65534, hard_age=65534, priority=0 actions=drop
  ```

  以vm发出arp报文为例：arp报文从br-int的7号口（qvod6acfbb9-49）进入，先查table=0的，然后查priority高的，然后再找匹配规则，于是找到了 `cookie=0x9c869d9a8de98c57, duration=147238.133s, table=0, n_packets=50, n_bytes=2100, idle_age=901, hard_age=65534, priority=10,arp,in_port=7 actions=resubmit(,24)` ，执行的动作是转到table=24中处理，进而找到 `cookie=0x9c869d9a8de98c57, duration=147238.143s, table=24, n_packets=50, n_bytes=2100, idle_age=901, hard_age=65534, priority=2,arp,in_port=7,arp_spa=10.0.0.3 actions=NORMAL` 可以看出，flow entry遇到从7口进入的arp，并且arp来源（arp_spa）是10.0.0.3的，就放行。

  再以vm发出icmp报文为例：icmp报文从br-int的7号口（qvod6acfbb9-49）进入，先查table=0的，然后查priority高的，然后再找匹配规则，于是找到了 `cookie=0x9c869d9a8de98c57, duration=148781.895s, table=0, n_packets=7722, n_bytes=852318, idle_age=40, hard_age=65534, priority=0 actions=NORMAL` ，于是放行，然后报文根据mac地址，转发给了4号口，进而发到了router的10.0.0.1的qr-79cbf0f1-a2插口。

- `ip netns exec qrouter-f157f1eb-d257-4637-b3ae-79febebbf102 ip addr` 查看router namespace中有哪些if

  ```
  sudo ip netns exec qrouter-f157f1eb-d257-4637-b3ae-79febebbf102 ip addr
  1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN
      link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
      inet 127.0.0.1/8 scope host lo
         valid_lft forever preferred_lft forever
      inet6 ::1/128 scope host
         valid_lft forever preferred_lft forever
  37: qr-79cbf0f1-a2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN
      link/ether fa:16:3e:0e:c4:6d brd ff:ff:ff:ff:ff:ff
      inet 10.0.0.1/24 brd 10.0.0.255 scope global qr-79cbf0f1-a2
         valid_lft forever preferred_lft forever
      inet6 fe80::f816:3eff:fe0e:c46d/64 scope link
         valid_lft forever preferred_lft forever
  38: qg-6b71d14f-27: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN
      link/ether fa:16:3e:ef:bb:b2 brd ff:ff:ff:ff:ff:ff
      inet 172.17.140.201/24 brd 172.17.140.255 scope global qg-6b71d14f-27
         valid_lft forever preferred_lft forever
      inet6 2001:db8::3/64 scope global
         valid_lft forever preferred_lft forever
      inet6 fe80::f816:3eff:feef:bbb2/64 scope link
         valid_lft forever preferred_lft forever
  ```

  ```
  sudo ip netns exec qrouter-f157f1eb-d257-4637-b3ae-79febebbf102 route -n
  Kernel IP routing table
  Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
  0.0.0.0         172.17.140.90   0.0.0.0         UG    0      0        0 qg-6b71d14f-27
  10.0.0.0        0.0.0.0         255.255.255.0   U     0      0        0 qr-79cbf0f1-a2
  172.17.140.0    0.0.0.0         255.255.255.0   U     0      0        0 qg-6b71d14f-27  
  ```

  ```
  sudo ovs-ofctl show br-ex
  OFPT_FEATURES_REPLY (xid=0x2): dpid:0000005056b4c237
  n_tables:254, n_buffers:256
  capabilities: FLOW_STATS TABLE_STATS PORT_STATS QUEUE_STATS ARP_MATCH_IP
  actions: output enqueue set_vlan_vid set_vlan_pcp strip_vlan mod_dl_src mod_dl_dst mod_nw_src mod_nw_dst mod_nw_tos mod_tp_src mod_tp_dst
   1(eth0): addr:00:50:56:b4:c2:37
       config:     0
       state:      0
       current:    1GB-FD COPPER AUTO_NEG
       advertised: 10MB-HD 10MB-FD 100MB-HD 100MB-FD 1GB-FD COPPER AUTO_NEG
       supported:  10MB-HD 10MB-FD 100MB-HD 100MB-FD 1GB-FD COPPER AUTO_NEG
       speed: 1000 Mbps now, 1000 Mbps max
   2(phy-br-ex): addr:c2:85:db:67:32:20
       config:     0
       state:      0
       speed: 0 Mbps now, 0 Mbps max
   LOCAL(br-ex): addr:00:50:56:b4:c2:37
       config:     0
       state:      0
       speed: 0 Mbps now, 0 Mbps max
  OFPT_GET_CONFIG_REPLY (xid=0x4): frags=normal miss_send_len=0  
  ```

  ```
  sudo ovs-ofctl dump-flows br-ex
  NXST_FLOW reply (xid=0x4):
   cookie=0x0, duration=151116.933s, table=0, n_packets=4105, n_bytes=339647, idle_age=41, hard_age=65534, priority=4,in_port=2,dl_vlan=2 actions=strip_vlan,NORMAL
   cookie=0x0, duration=151147.726s, table=0, n_packets=3381, n_bytes=484696, idle_age=3267, hard_age=65534, priority=2,in_port=2 actions=drop
   cookie=0x0, duration=151147.820s, table=0, n_packets=1301567, n_bytes=484025430, idle_age=0, hard_age=65534, priority=0 actions=NORMAL  
  ```

![](/public/images/flow.png)


总结
----

因此： 从vm中ping 172.17.140.140，从vm到外网的数据包流向：

- vm知道目的ip不属于本网段，于是arp广播，询问网关10.0.0.1的mac地址
- arp包经过bridge，进而进入br-int的7号口，进而从4号口出去，达到router的10.0.0.1端口，随后mac地址原路返回
- vm将10.0.0.1的mac地址写入frame头，ip报文中的目标ip仍然是外网地址172.17.140.140
- 达到router后，router根据自己的路由表来判断该ip报文的下一跳是谁
- router发现172.17.140.140和自己的ip 172.17.140.201是同一网段的，于是直接arp广播，询问172.17.140.140的mac地址
- arp报文经过了br-ex，从eth0出去了
