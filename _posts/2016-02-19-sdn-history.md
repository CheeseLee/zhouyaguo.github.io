---
layout: post
title: "sdn历史"
comments: true
categories:
- openstack
---

_谈论一个事物，不能脱离它的历史。_

Stanford大学有个[Clean Slate](http://cleanslate.stanford.edu/)项目，项目最终目的是重新发明Internet，改变难以优化的现有网络架构。

2006年，[Martin Casado](http://yuba.stanford.edu/~casado/)还是Stanford的一名学生，领导了一个关于网络安全与管理的项目
[Ethane](http://yuba.stanford.edu/ethane/)，该项目试图通过一个集中的控制器（控制器实现网络主机认证，ip分配，生成流表，是整个网络的控制决策层），让网络管理员可以方便的定义网络流的安全策略，并将这些策略应用到网络设备中，
从而实现对整个网络通讯的安全控制。Ethane实现了交换机和controller的大部分功能，奠定了openFlow的基础。

2007年，Martin Casado有个疯狂的想法：如何协调全球的网络的流量，改变整个互联网的架构？
加州伯克利的Scott Shenker当时认为没有人会买账，因为大家更关心企业级的网络如何控制，所以他们的第一篇论文Ethane: **Taking control of the enterprise** 的内容就是将Martin的想法在企业中实现，论文中将网络分为一个物理层和应用层。后来他们意识到在控制层面应该提供一个更为通用的接口，这才是SDN诞生。

Martin和它的导师Nick McKeown发现，如果将Ethane设计更一般化，将传统的网络设备的数据转发（data plane）
和路由控制（control plane）分开，通过集中的控制器（controller）使用标准的接口对各种网络设备进行管理，将会推动网络的革新，于是他们提出openFlow的概念。

2007年， Martin Casado、 Nick McKeown、 Scott Shenker成立Nicira公司.

2008年，Martin Casado博士毕业。

2008年，Nick McKeown和加州伯克利的Scott Shenker等人在ACM SIGCOMM发表了论文：
[OpenFlow: Enabling Innovation in Campus Networks](http://www.openflow.org/documents/openflow-wp-latest.pdf)
介绍了openFlow的概念。openFlow为SDN提供了标准，各个厂商都可以在这个标准上作为玩家。
网络设备上维护流表（flow table），数据按照流表进行转发，而流表的生成，维护，配置则由controller来管理。
在OpenFlow网络中，将不再区分路由器和交换机，而是统称为OpenFlow交换机。
在这种架构下，网络的运行维护只需要通过软件的更新来实现网络功能的升级，网络管理者无需再配置特定的厂商设备，
从而加速了网络部署的周期。

2009年，Kate Greene在[TechnologyReview](http://www.technologyreview.com/)中提出SDN的概念：如果将网络设备看做是被管理的资源，
那么参照操作系统的原理，可以将底层网络设备具体细节抽象化，并为上层应用提供统一的API，这样，用户就可以
开发各种应用程序，通过软件来定义网络拓扑，以满足对网络资源的不同需求，而不需要关心网络的物理拓扑结构。

2009年，Martin Casado创立Open vSwitch项目，为云平台例如openstack提供网络虚拟化。SDN不等于网络虚拟化。

2009年，Martin Casado用python语言创立了[nox](https://en.wikipedia.org/wiki/Nox_(platform))项目，nox是第一个OpenFlow controller。

2011年，Nick McKeown和Scott Shenker成立了非盈利组织ONF（Open Networking Foundation），
致力于SDN和openFlow的标准化和规范维护，还发布了SDN白皮书： [Software Defined Networking：The New Norm for Networks](https://www.opennetworking.org/images/stories/downloads/white-papers/wp-sdn-newnorm.pdf)

2011年随后，第一届开放网络峰会（OpenNetworking Summit）召开，为SDN和OpenFlow在业界做了推广。

2012年，[第二届开放网络峰会](http://opennetsummit.org/speakers.html)，_OpenFlow@Google_ 的演讲中宣布了Google在全球的DataCenter大规模使用了
OpenFlow/SDN，起到了示范作用，从而证明了OpenFlow技术成熟了。

2012年7月，Nicira被vmware以12.6亿美元收购，一年以后，vmware的NSX网络虚拟化方案出现。

2012年，Cisco沉寂多时后，宣布了ONE（open network environment）计划，ONE包括One Platform Kit，
为开发人员提供Cisco的所有设备的API。

2013年，cisco和ibm联合众多厂商成立OpenDaylight
