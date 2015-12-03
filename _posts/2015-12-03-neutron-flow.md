---
layout: post
title: "[openstack] [neutron] 数据包流向：vm->外网"
comments: true
categories:
- openstack
---

说明： 当连某ip时(不管在不在一个子网内)，由于会发arp广播，来获取对应的mac地址。所以只需要关注端口和谁连就行了。

通过一系列的命令，可以追踪从vm到外网的网络包传输路径：

- 查看vm vif： `cat /opt/stack/data/nova/instances/4c387232-8c28-4106-8440-262421f9a44b/libvirt.xml`

  ```  
  <interface type="bridge">
    <mac address="fa:16:3e:ef:33:9f"/>
    <model type="virtio"/>
    <driver name="qemu"/>
    <source bridge="qbr0a290825-8c"/>  --tap0a290825-8c这个接口是在qbr0a290825-8c bridge上
    <target dev="tap0a290825-8c"/>     --vm连到tap0a290825-8c上了
  </interface>  
  ```

  ```
  [root@centos7-openstack-L-test nova]# ifconfig tap0a290825-8c
  tap0a290825-8c: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
          inet6 fe80::fc16:3eff:feef:339f  prefixlen 64  scopeid 0x20<link>
          ether fe:16:3e:ef:33:9f  txqueuelen 500  (Ethernet)
          RX packets 123  bytes 12674 (12.3 KiB)
          RX errors 0  dropped 0  overruns 0  frame 0
          TX packets 223  bytes 21003 (20.5 KiB)
          TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
  ```
- 查看linux bridge： `brctl show` 。bridge的两个口，一个连vm，一个连ovs

  ```
  bridge name       bridge id		       STP enabled	interfaces
  qbr0a290825-8c		8000.1af8e02c4024	      no		qvb0a290825-8c
  							                                   tap0a290825-8c
  ```
- `ethtool -S qvb0a290825-8c` 找出peer

  ```
  ethtool -S qvb0a290825-8c
  NIC statistics:
       peer_ifindex: 26
  ```

- `ip addr` 找到peer是 qvo0a290825-8c

  ```
  26: qvo0a290825-8c: <BROADCAST,MULTICAST,PROMISC,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast master ovs-system state UP qlen 1000
      link/ether d2:4c:e0:91:db:2a brd ff:ff:ff:ff:ff:ff
      inet6 fe80::d04c:e0ff:fe91:db2a/64 scope link
         valid_lft forever preferred_lft forever
  ```

- `ovs-vsctl show`，找出qvo0a290825-8c，属于vlan4，位于br-int，br-int与br-ex通过veth pair连接

  ```
  ovs-vsctl show
  fa6c9931-a88f-4d59-b1ec-a4b56d3f8d28
      Bridge br-ex
          Port phy-br-ex
              Interface phy-br-ex
                  type: patch
                  options: {peer=int-br-ex}
          Port br-ex
              Interface br-ex
                  type: internal
          Port "ens32"
              Interface "ens32"
      Bridge br-int
          fail_mode: secure
          Port "qr-dfd69133-2b"
              tag: 6
              Interface "qr-dfd69133-2b"
                  type: internal
          Port "qr-1874c50a-46"
              tag: 4
              Interface "qr-1874c50a-46"
                  type: internal
          Port "tap03478a28-d2"
              tag: 4
              Interface "tap03478a28-d2"
                  type: internal
          Port patch-tun
              Interface patch-tun
                  type: patch
                  options: {peer=patch-int}
          Port "tap899e627f-da"
              tag: 6
              Interface "tap899e627f-da"
                  type: internal
          Port "qvo0a290825-8c"
              tag: 4
              Interface "qvo0a290825-8c"
          Port br-int
              Interface br-int
                  type: internal
          Port "qvo9d203880-bf"
              tag: 6
              Interface "qvo9d203880-bf"
          Port int-br-ex
              Interface int-br-ex
                  type: patch
                  options: {peer=phy-br-ex}
          Port "qg-5d0e87ec-fc"
              tag: 5
              Interface "qg-5d0e87ec-fc"
                  type: internal
          Port "qvo18d4ad09-42"
              tag: 4
              Interface "qvo18d4ad09-42"
      Bridge br-tun
          fail_mode: secure
          Port br-tun
              Interface br-tun
                  type: internal
          Port patch-int
              Interface patch-int
                  type: patch
                  options: {peer=patch-tun}
      ovs_version: "2.3.1"

  ```

- `ovs-ofctl dump-flows br-int` 查看ovs的流表处理

  ```
  [root@centos7-openstack-L-test nova]# ovs-ofctl dump-flows br-int
  NXST_FLOW reply (xid=0x4):
   cookie=0xa4af656188366856, duration=157437.995s, table=0, n_packets=29705, n_bytes=3450286, idle_age=1, hard_age=65534, priority=0 actions=NORMAL
   cookie=0xa4af656188366856, duration=157437.829s, table=0, n_packets=4974, n_bytes=652322, idle_age=28285, hard_age=65534, priority=2,in_port=1 actions=drop
   cookie=0xa4af656188366856, duration=9381.477s, table=0, n_packets=20, n_bytes=840, idle_age=803, priority=10,arp,in_port=14 actions=resubmit(,24)
   cookie=0xa4af656188366856, duration=27147.069s, table=0, n_packets=65, n_bytes=2730, idle_age=5, priority=10,arp,in_port=11 actions=resubmit(,24)
   cookie=0xa4af656188366856, duration=9561.432s, table=0, n_packets=23, n_bytes=966, idle_age=803, priority=10,arp,in_port=12 actions=resubmit(,24)
   cookie=0xa4af656188366856, duration=9381.496s, table=0, n_packets=0, n_bytes=0, idle_age=9381, priority=10,icmp6,in_port=14,icmp_type=136 actions=resubmit(,24)
   cookie=0xa4af656188366856, duration=27147.089s, table=0, n_packets=0, n_bytes=0, idle_age=27147, priority=10,icmp6,in_port=11,icmp_type=136 actions=resubmit(,24)
   cookie=0xa4af656188366856, duration=9561.453s, table=0, n_packets=0, n_bytes=0, idle_age=9561, priority=10,icmp6,in_port=12,icmp_type=136 actions=resubmit(,24)
   cookie=0xa4af656188366856, duration=28285.537s, table=0, n_packets=150147, n_bytes=24405722, idle_age=0, priority=3,in_port=1,vlan_tci=0x0000 actions=mod_vlan_vid:5,NORMAL
   cookie=0xa4af656188366856, duration=157437.987s, table=23, n_packets=0, n_bytes=0, idle_age=65534, hard_age=65534, priority=0 actions=drop
   cookie=0xa4af656188366856, duration=157437.978s, table=24, n_packets=0, n_bytes=0, idle_age=65534, hard_age=65534, priority=0 actions=drop
   cookie=0xa4af656188366856, duration=27147.079s, table=24, n_packets=65, n_bytes=2730, idle_age=5, priority=2,arp,in_port=11,arp_spa=10.0.0.3 actions=NORMAL
   cookie=0xa4af656188366856, duration=9381.486s, table=24, n_packets=20, n_bytes=840, idle_age=803, priority=2,arp,in_port=14,arp_spa=10.0.1.3 actions=NORMAL
   cookie=0xa4af656188366856, duration=9561.442s, table=24, n_packets=23, n_bytes=966, idle_age=803, priority=2,arp,in_port=12,arp_spa=10.0.0.4 actions=NORMAL
   cookie=0xa4af656188366856, duration=9381.505s, table=24, n_packets=0, n_bytes=0, idle_age=9381, priority=2,icmp6,in_port=14,icmp_type=136,nd_target=fe80::f816:3eff:fe01:fed0 actions=NORMAL
   cookie=0xa4af656188366856, duration=9561.463s, table=24, n_packets=0, n_bytes=0, idle_age=9561, priority=2,icmp6,in_port=12,icmp_type=136,nd_target=fe80::f816:3eff:feef:339f actions=NORMAL
   cookie=0xa4af656188366856, duration=27147.099s, table=24, n_packets=0, n_bytes=0, idle_age=27147, priority=2,icmp6,in_port=11,icmp_type=136,nd_target=fe80::f816:3eff:fe6f:216 actions=NORMAL
  ```

- `ovs-ofctl dump-ports-desc br-int` 查看port对应的编号
  ```
  [root@centos7-openstack-L-test nova]# ovs-ofctl dump-ports-desc br-int
  OFPST_PORT_DESC reply (xid=0x2):
   1(int-br-ex): addr:a2:ed:17:92:32:f4
       config:     0
       state:      0
       speed: 0 Mbps now, 0 Mbps max
   2(patch-tun): addr:a6:a4:19:66:c1:d1
       config:     0
       state:      0
       speed: 0 Mbps now, 0 Mbps max
   8(tap03478a28-d2): addr:00:00:00:00:00:00
       config:     PORT_DOWN
       state:      LINK_DOWN
       speed: 0 Mbps now, 0 Mbps max
   9(qg-5d0e87ec-fc): addr:00:00:00:00:00:00
       config:     PORT_DOWN
       state:      LINK_DOWN
       speed: 0 Mbps now, 0 Mbps max
   10(qr-1874c50a-46): addr:00:00:00:00:00:00
       config:     PORT_DOWN
       state:      LINK_DOWN
       speed: 0 Mbps now, 0 Mbps max
   11(qvo18d4ad09-42): addr:36:42:63:84:05:e5
       config:     0
       state:      0
       current:    10GB-FD COPPER
       speed: 10000 Mbps now, 0 Mbps max
   12(qvo0a290825-8c): addr:d2:4c:e0:91:db:2a
       config:     0
       state:      0
       current:    10GB-FD COPPER
       speed: 10000 Mbps now, 0 Mbps max
   13(tap899e627f-da): addr:00:00:00:00:00:00
       config:     PORT_DOWN
       state:      LINK_DOWN
       speed: 0 Mbps now, 0 Mbps max
   14(qvo9d203880-bf): addr:a6:9e:2c:0b:ee:2e
       config:     0
       state:      0
       current:    10GB-FD COPPER
       speed: 10000 Mbps now, 0 Mbps max
   15(qr-dfd69133-2b): addr:00:00:00:00:00:00
       config:     PORT_DOWN
       state:      LINK_DOWN
       speed: 0 Mbps now, 0 Mbps max
   LOCAL(br-int): addr:32:db:7c:98:c6:4a
       config:     PORT_DOWN
       state:      LINK_DOWN
       speed: 0 Mbps now, 0 Mbps max
  ```

  `cookie=0xa4af656188366856, duration=28285.537s, table=0, n_packets=150147, n_bytes=24405722, idle_age=0, priority=3,in_port=1,vlan_tci=0x0000 actions=mod_vlan_vid:5,NORMAL` 含义是： 从1号port（int-br-ex）进入的包，br-int就将包修改为vlan5

  ```
  [root@centos7-openstack-L-test nova]# ovs-ofctl dump-flows br-ex
  NXST_FLOW reply (xid=0x4):
   cookie=0x0, duration=161350.750s, table=0, n_packets=4875744, n_bytes=627654937, idle_age=0, hard_age=65534, priority=0 actions=NORMAL
   cookie=0x0, duration=32198.391s, table=0, n_packets=1296, n_bytes=106714, idle_age=30, priority=4,in_port=2,dl_vlan=5 actions=strip_vlan,NORMAL
   cookie=0x0, duration=161350.664s, table=0, n_packets=24552, n_bytes=3099663, idle_age=9, hard_age=65534, priority=2,in_port=2 actions=drop
  ```

  ```
  [root@centos7-openstack-L-test nova]# ovs-ofctl dump-ports-desc br-ex
  OFPST_PORT_DESC reply (xid=0x2):
   1(ens32): addr:00:50:56:b4:43:8f
       config:     0
       state:      0
       current:    1GB-FD COPPER AUTO_NEG
       advertised: 10MB-HD 10MB-FD 100MB-HD 100MB-FD 1GB-FD COPPER AUTO_NEG
       supported:  10MB-HD 10MB-FD 100MB-HD 100MB-FD 1GB-FD COPPER AUTO_NEG
       speed: 1000 Mbps now, 1000 Mbps max
   2(phy-br-ex): addr:46:0f:6c:e3:c6:7e
       config:     0
       state:      0
       speed: 0 Mbps now, 0 Mbps max
   LOCAL(br-ex): addr:00:50:56:b4:43:8f
       config:     0
       state:      0
       speed: 0 Mbps now, 0 Mbps max
  ```

  ` cookie=0x0, duration=32198.391s, table=0, n_packets=1296, n_bytes=106714, idle_age=30, priority=4,in_port=2,dl_vlan=5 actions=strip_vlan,NORMAL` 含义是： 从2号port（phy-br-ex）进入的包，并且是vlan5，就把vlan去掉。

总结
----

因此： 从vm到外网的数据包流向：

vm -> tap0a290825-8c（bridge上的port，给vm连的） -> qvb0a290825-8c -> br-int上的qvo0a290825-8c(打tag 4) -> br-int上的int-br-ex（打vlan5） ->
br-ex上的phy-br-ex（去掉vlan5 tag） -> ens32
