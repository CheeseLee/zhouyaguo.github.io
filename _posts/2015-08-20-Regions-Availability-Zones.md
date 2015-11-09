---
layout: post
title: "[openstack] Region和Availability Zone的概念"
comments: true
categories:
- openstack
---

Region和Availability Zone的概念来源于AWS，在openstack中也有同样的概念。

Region在地理位置上是个独立的区域，每个Region之间是完全独立的。例如北京机房和上海机房是两个不同的Region

每个Region中又可以分成多个Availability Zones，例如 production zone， test zone等。

![](/public/images/region.png)

Reference
=========

[http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-regions-availability-zones.html](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-regions-availability-zones.html)
