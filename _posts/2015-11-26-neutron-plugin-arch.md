---
layout: post
title: "[openstack] [neutron] Neutron plugin体系"
comments: true
categories:
- openstack
---


neutron设计了一个plugin体系，在`NeutronPluginBaseV2`类中定义了plugin的API，
主要就是network，subnet，port三种资源的CRUD操作

```
NeutronPluginBaseV2  -->定义抽象方法
       ^
       |
NeutronDbPluginV2  -->负责network，subnet，port三种资源的表CRUD操作
       ^
       |
    Ml2Plugin      -->在db操作之后，接着调用extension_manager、type_manager、mechanism_manager
```

ML2 plugin architecture
------------------------
从Havana版本开始，openvswitch和linuxbridge plugins被Modular Layer 2 (ML2) plugin 取代

ML2用 *type drivers* 来支持多种网络技术，用 *mechanism drivers* 来配置网络

长远的计划是将所有厂商相关的plugin都转换成type和mechanism drivers

ML2的处理过程
-----------

- create_network
  - db操作create\_network
  - extension_manager.process\_create\_network
  - type_manager.create\_network\_segments
  - mechanism_manager.create\_network\_precommit
  - db commit
  - mechanism_manager.create\_network\_postcommit
- create_subnet
  - db操作create_subnet
  - extension_manager.process\_create\_subnet
  - mechanism_manager.create\_subnet\_precommit
  - db commit
  - mechanism_manager.create\_subnet\_postcommit
- create_port
  - db操作create_port
  - extension_manager.process\_create\_port
  - 处理port biding
  - mechanism_manager.create\_port\_precommit
  - db commit
  - mechanism_manager.create\_port\_postcommit

`其中只有create_network才用到type_manager`

network type种类
-----------------

neutron支持每个tenant拥有多个private networks，并且允许ip overlap，即允许多个tenant使用相同的private addresses

用户是在project的范围内创建tenant network，与其他project是隔离的。

按网络的隔离方法的不同，分为以下几种方式：

- flat
  - 所有instance在同一network中，没有网络隔离。
- vlan
  - 用户创建多个tenant network来对应物理网络中的vlan，instances可以直接和外部同一L2 vlan中的其他server通信，包括servers，firewalls，load balancers等
- gre and vxlan
  - gre和vxlan是用来为instances创建overlay networks，这种网络模式需要有一个router，networks内部的instance要经过router出去，外部可以通过floating ip来进入内部
- geneve

ml2 extension种类
-----------------

位于`neutron/neutron/extensions`，提供各种除了network，subnet，port之外的其他服务，具体例如：

- dns
- qos
- rbac
- SecurityGroup
- metering
- dvr
- quotas
- dhcp
- route
- mtu


ml2 mechanism种类
-----------------

不同厂商提供不同的mechanism driver（源码位于`neutron/neutron/plugins/ml2/drivers/`）

- brocade
- freescale
- hyperv
- ibm
- l2pop
- linuxbridge
- mech_bigswitch
- mech_sriov
- mlnx
- ofagent
- opendaylight
- openvswitch
- ovsvapp
- IBM SDNVE
- L2Population

动手写plugin
-----------

步骤：

- neutron.egg-info/entry\_points.txt的[neutron.core\_plugins]部分，要定义一下，例如：midonet = neutron.plugins.midonet.plugin:MidonetPluginV2
- /etc/neutron.conf中设置core_plugin
- neutron/plugins写具体的plugin厂商实现

如果厂商除了network、subnet、port之外，还有一些额外的功能，比如router，firewall等，怎么办？就需要用到extension


动手写extension
---------------

步骤：

1. 写一个RESOURCE\_ATTRIBUTE_MAP，类似neutron/api/v2/attributes.py中的NETWORKS的写法
1. 写一个类，继承extensions.ExtensionDescriptor
1. /etc/neutron/neutron.conf中配置api\_extensions\_path
1. 在自己的plugin中写supported\_extension\_aliases = ['my-extensions']，以表明自己的plugin支持该扩展
1. 如果该extension需要db的话，则参照NeutronDbPluginV2来写数据库操作
1. 在neutronclient/shell.py中添加CLI命令参数
1. 在neutronclient/neutron/v2\_0/下新建自己的文件夹，新建5个类，分别实现List/Show/Create/Delete/Update，这些class负责将命令转化为API call，最后会被neutron后台的plugin controller处理


源码中的注释
-----------

- TypeDriver

  ```
  Define stable abstract interface for ML2 type drivers.

  ML2 type drivers each support a specific network_type for provider
  and/or tenant network segments. Type drivers must implement this
  abstract interface, which defines the API by which the plugin uses
  the driver to manage the persistent type-specific resource
  allocation state associated with network segments of that type.

  Network segments are represented by segment dictionaries using the
  NETWORK_TYPE, PHYSICAL_NETWORK, and SEGMENTATION_ID keys defined
  above, corresponding to the provider attributes.  Future revisions
  of the TypeDriver API may add additional segment dictionary
  keys. Attributes not applicable for a particular network_type may
  either be excluded or stored as None.
  ```

- MechanismDriver

  ```
  Define stable abstract interface for ML2 mechanism drivers.

  A mechanism driver is called on the creation, update, and deletion
  of networks and ports. For every event, there are two methods that
  get called - one within the database transaction (method suffix of
  _precommit), one right afterwards (method suffix of _postcommit).

  Exceptions raised by methods called inside the transaction can
  rollback, but should not make any blocking calls (for example,
  REST requests to an outside controller). Methods called after
  transaction commits can make blocking external calls, though these
  will block the entire process. Exceptions raised in calls after
  the transaction commits may cause the associated resource to be
  deleted.

  Because rollback outside of the transaction is not done in the
  update network/port case, all data validation must be done within
  methods that are part of the database transaction.
  ```

- ExtensionDriver

  ```
  Define stable abstract interface for ML2 extension drivers.

  An extension driver extends the core resources implemented by the
  ML2 plugin with additional attributes. Methods that process create
  and update operations for these resources validate and persist
  values for extended attributes supplied through the API. Other
  methods extend the resource dictionaries returned from the API
  operations with the values of the extended attributes.
  ```

Qos plugin
----------

server端设计

  - neutron.extensions.qos 定义api
  - neutron.services.qos.qos\_plugin _实现qos extension，具体处理rule_
  - neutron.services.qos.notification\_drivers.manager _负责把notifications传给notifications driver_
  - neutron.services.qos.notification\_drivers.qos_base _接收event，并对create, update, delete操作做处理_
  - neutron.services.qos.notification\_drivers.message_queue _通过rpc来update agent_
  - neutron.core\_extensions.base _实现core resource (port/network) extensions的接口_
  - neutron.core\_extensions.qos _qos的core resource extension_
  - neutron.plugins.ml2.extensions.qos _是ml2 extension driver_

agent端设计

  - //TODO

server端配置

  - service_plugins加上qos
  - [qos]中设置notification_drivers，默认是message_queue
  - [ml2]中设置extension_drivers，加上qos

agent端配置

  - [agent]加qos
