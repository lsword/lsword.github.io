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

bridge模块的组要功能是进行初始化操作，主要包括创建网桥、配置iptable、注册网络管理函数。

#### 3.1 创建网桥

通过netlink方式创建网桥设备，具体创建方式后续补充。

#### 3.2 配置iptable

将iptables配置为以下内容：

~~~
root@ubuntu:/home/paas# iptables -S
-P INPUT ACCEPT
-P FORWARD ACCEPT
-P OUTPUT ACCEPT
-A FORWARD -o docker0 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
-A FORWARD -i docker0 ! -o docker0 -j ACCEPT
-A FORWARD -i docker0 -o docker0 -j ACCEPT

root@ubuntu:/home/paas# iptables -S -t nat
-P PREROUTING ACCEPT
-P INPUT ACCEPT
-P OUTPUT ACCEPT
-P POSTROUTING ACCEPT
-N DOCKER
-A PREROUTING -m addrtype --dst-type LOCAL -j DOCKER
-A OUTPUT ! -d 127.0.0.0/8 -m addrtype --dst-type LOCAL -j DOCKER
-A POSTROUTING -s 172.17.0.0/16 ! -d 172.17.0.0/16 -j MASQUERADE
~~~

打开本机的ip转发，即将/proc/sys/net/ipv4/ip_forward设置为。.


#### 3.3 注册网络管理函数

注册以下四个函数，用于后续使用。

~~~
allocate_interface	分配IP地址（ipallocator实现）
release_interface	回收IP地址（ipallocator实现）
allocate_port		分配端口（portallocator实现）
link				连接容器（LinkContainers函数）
~~~

LinkContainers函数主要目的是配置iptables的nat规则，实现主机到容器的nat转换。

### 4 ipallocator

ipallocator主要对外提供两个函数：

~~~
RequestIP：分配IP
	如果未指定IP，则从IP地址池中分配一个IP。
	如果指定了IP，则检查IP释放可用。
ReleaseIP：释放IP
	释放一个已分配的IP。
~~~

两个关键变量：

~~~
allocatedIPs：已分配的IP
availableIPS：可用的IP
~~~

### 5 portallocator

在当前主机上分配一个端口。

端口范围：49153-65535

### 6 portmapper

portmapper主要通过配置iptables规则，实现针对特定协议的，从主机IP:PORT到容器的IP:PORT的消息转发配置。



[Docker之graphdriver]: http://lsword.github.io/2014/06/03.html
[thinprovisioned_volumes]: https://access.redhat.com/site/documentation/en-US/Red_Hat_Enterprise_Linux/6/html/Logical_Volume_Manager_Administration/thinprovisioned_volumes.html
[device-mapper]: http://device-mapper.com/
[thin-provisioning.txt]:https://github.com/torvalds/linux/blob/master/Documentation/device-mapper/thin-provisioning.txt