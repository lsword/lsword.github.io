---
layout: post
category: 云计算 虚拟化
tags: docker
description: 本文介绍docker容器中实现容器网络配置管理的networkdriver
---

### 1、背景

本文中涉及的源码基于docker 0.11.1版本系统。
本文中涉及的测试环境基于ubuntu server 14.04及redhat6.5。

### 2、概述


docker的networkdriver是用来实现容器网络配置管理的，涉及到ip分配、端口分配、iptable规则配置等问题。

networkdriver与execdriver、graphdriver模式不一样，代码组织不是通过接口的方式来实现多种驱动以便提高兼容性。

networkdriver由bridge、ipallocator、portallocator、portmapper组成。

### 3 bridge

### 4 ipallocator

### 5 portallocator

### 6 portmapper


### 参考文档


[Docker之graphdriver]: http://lsword.github.io/2014/06/03.html
[thinprovisioned_volumes]: https://access.redhat.com/site/documentation/en-US/Red_Hat_Enterprise_Linux/6/html/Logical_Volume_Manager_Administration/thinprovisioned_volumes.html
[device-mapper]: http://device-mapper.com/
[thin-provisioning.txt]:https://github.com/torvalds/linux/blob/master/Documentation/device-mapper/thin-provisioning.txt