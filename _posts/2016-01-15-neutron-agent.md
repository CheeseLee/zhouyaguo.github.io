---
layout: post
title: "[openstack] [neutron] Neutron agent"
comments: true
categories:
- openstack
---


L2 agent
---------

OpenVSwitch L2 Agent

ovs-neutron-agent可以通过设置不同的网络拓扑，来实现tenant之间的隔离：

  - vlan
  - gre tunnel (https://www.rdoproject.org/networking/networking-in-too-much-detail/)
  - vxlan tunnel (是一种overlay技术，将frame放入udp， http://en.wikipedia.org/wiki/Virtual\_Extensible\_LAN)
  - Geneve tunnel (条件： kernel >= 3.18, OVS version >=2.4， centos7中默认是3.10.0)

agent使用br-tun，br-int，原因：

  - 让agent可以处理多种tunnel技术
  - segmentation和tenant isolation之间解耦
  - 兼容没有使用tunnel的OVS agents

所有vm的VIF都插入br-int

veth用来连接br-int上的vlan和物理linux bridge，期间会通过flow rule来add，modify，strip vlan tag


L3 agent
---------
