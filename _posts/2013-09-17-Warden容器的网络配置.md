---
layout: post
category: 云计算
tags: CloudFoundry 容器 虚拟化
description: warden容器的网络配置部分涉及到地址转换、路由配置、流量控制等linux网络方面的相关知识，本文通过一个实际的配置情况来对这部分内容进行说明。
---

### 术语
  1. warden主机：运行warden的主机，其上可以运行多个warden容器。
  2. warden容器：运行在warden主机上的虚拟主机，CloudFoundry的每个应用实例就运行在一个warden容器中。
  3. 应用实例：运行在warden容器中的一个应用。

### warden容器的网络配置

#### ifconfig

		lo    Link encap:Local Loopback
    	      inet addr:127.0.0.1  Mask:255.0.0.0
        	  inet6 addr: ::1/128 Scope:Host
	          UP LOOPBACK RUNNING  MTU:16436  Metric:1
    	      RX packets:0 errors:0 dropped:0 overruns:0 frame:0
        	  TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
	          collisions:0 txqueuelen:0
    	      RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
    	      
		w-1762ge8c9k9-1 Link encap:Ethernet  HWaddr ba:5d:57:99:66:8c
    	      inet addr:10.254.0.14  Bcast:10.254.0.15  Mask:255.255.255.252
	          inet6 addr: fe80::b85d:57ff:fe99:668c/64 Scope:Link
    	      UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
        	  RX packets:62 errors:0 dropped:0 overruns:0 frame:0
	          TX packets:60 errors:0 dropped:0 overruns:0 carrier:0
    	      collisions:0 txqueuelen:1000
        	  RX bytes:17789 (17.7 KB)  TX bytes:6204 (6.2 KB)

说明：

  * 有一个名为w-1762ge8c9k9-1的以太网卡，其中1762ge8c9k9是容器的ID。
  * 从子网掩码可以看出子网中包含4个IP地址，从10.254.0.12到10.254.0.15。
  * 子网地址为10.254.0.12
  * 子网广播地址为10.254.0.15
  * 网卡IP地址为10.254.0.14

#### route
	Kernel IP routing table
	Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
	default         10.254.0.13     0.0.0.0         UG    0      0        0 w-1762ge8c9k9-1
	10.254.0.12     *               255.255.255.252 U     0      0        0 w-1762ge8c9k9-1

说明：

  * 网关地址为10.254.0.13
  
#### tc
	qdisc pfifo_fast 0: dev w-1762ge8c9k9-1 root refcnt 2 bands 3 priomap  1 2 2 2 1 2 0 0 1 1 1 1 1 1 1 1
	 Sent 6372 bytes 63 pkt (dropped 0, overlimits 0 requeues 0)
	 backlog 0b 0p requeues 0

### warden主机的网络配置

#### ifconfig

	eth0      Link encap:以太网  硬件地址 00:50:56:8f:49:37
    	      inet 地址:10.1.59.242  广播:10.1.59.255  掩码:255.255.252.0
	          inet6 地址: fe80::250:56ff:fe8f:4937/64 Scope:Link
        	  UP BROADCAST RUNNING MULTICAST  MTU:1500  跃点数:1
	          接收数据包:487571 错误:0 丢弃:0 过载:0 帧数:0
    	      发送数据包:871859 错误:0 丢弃:0 过载:0 载波:0
        	  碰撞:0 发送队列长度:1000
	          接收字节:97674356 (97.6 MB)  发送字节:501385036 (501.3 MB)

	lo        Link encap:本地环回
    	      inet 地址:127.0.0.1  掩码:255.0.0.0
        	  inet6 地址: ::1/128 Scope:Host
	          UP LOOPBACK RUNNING  MTU:16436  跃点数:1
    	      接收数据包:18852195 错误:0 丢弃:0 过载:0 帧数:0
        	  发送数据包:18852195 错误:0 丢弃:0 过载:0 载波:0
        	  碰撞:0 发送队列长度:0
        	  接收字节:11960627808 (11.9 GB)  发送字节:11960627808 (11.9 GB)
        	  
	w-1762ge8ck9-0 Link 	encap:以太网  硬件地址 12:21:c4:1d:8a:e8
	          inet 地址:10.254.0.13  广播:10.254.0.15  掩码:255.255.255.252
    	      inet6 地址: fe80::1021:c4ff:fe1d:8ae8/64 Scope:Link
        	  UP BROADCAST RUNNING MULTICAST  MTU:1500  跃点数:1
	          接收数据包:63 错误:0 丢弃:0 过载:0 帧数:0
    	      发送数据包:65 错误:0 丢弃:0 过载:0 载波:0
        	  碰撞:0 发送队列长度:1000
	          接收字节:6372 (6.3 KB)  发送字节:18034 (18.0 KB)
说明：

  * eth0为warden主机的物理网卡。
  * w-1762ge8c9k9-0为warden主机的虚拟网卡，其中1762ge8c9k9是容器的ID。
  * w-1762ge8c9k9-0的IP为10.254.0.13，是warden容器中网卡对应的网关地址。

#### route

	内核 IP 路由表
	目标            网关            子网掩码           标志  跃点    引用      使用 接口
	default         10.1.56.1       0.0.0.0         UG    100    0        0    eth0
	10.1.56.0       *               255.255.252.0   U     0      0        0    eth0
	10.254.0.12     *               255.255.255.252 U     0      0        0    w-1762ge8c9k9-0

#### tc

	qdisc pfifo_fast 0: dev eth0 root refcnt 2 bands 3 priomap  1 2 2 2 1 2 0 0 1 1 1 1 1 1 1 1
	 Sent 336397677 bytes 873959 pkt (dropped 0, overlimits 0 requeues 0)
	 backlog 0b 0p requeues 0
	 
	qdisc pfifo_fast 0: dev w-1762ge8c9k9-0 root refcnt 2 bands 3 priomap  1 2 2 2 1 2 0 0 1 1 1 1 1 1 1 1
	 Sent 18279 bytes 68 pkt (dropped 0, overlimits 0 requeues 0)
	 backlog 0b 0p requeues 0
	 
#### iptables listrules

	-P INPUT ACCEPT
	-P FORWARD ACCEPT
	-P OUTPUT ACCEPT
	-N warden-default
	-N warden-forward
	-N warden-instance-1762ge8c9k9
	-A FORWARD -i w-+ -j warden-forward
	-A warden-default -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
	-A warden-forward -i w-1762ge8c9k9-0 -g warden-instance-1762ge8c9k9
	-A warden-forward -j DROP
	-A warden-instance-1762ge8c9k9 -g warden-default

说明：

  * 来自设备w-1762ge8c9k9-0的请求都应用规则：-m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT。
  * -m conntrack，为每一个经过网络堆栈的数据包，生成一个新的连接记录项 （Connection entry）。此后，所有属于此连接的数据包都被唯一地分配给这个连接，并标识连接的状态。连接跟踪是防火墙模块的状态检测的基础，同时也是地址转换中实 现SNAT和DNAT的前提。
  * --ctstate RELATED,ESTABLISHED -j ACCEPT 允许所有已经建立的和相关的连接。
   
#### iptables nat
	Chain PREROUTING (policy ACCEPT)
	target     prot opt source               destination
	warden-prerouting  all  --  anywhere             anywhere

	Chain INPUT (policy ACCEPT)
	target     prot opt source               destination

	Chain OUTPUT (policy ACCEPT)
	target     prot opt source               destination
	warden-prerouting  all  --  anywhere             anywhere

	Chain POSTROUTING (policy ACCEPT)
	target     prot opt source               destination
	warden-postrouting  all  --  anywhere             anywhere

	Chain warden-instance-1762ge8c9k9 (1 references)
	target     prot opt source               destination
	DNAT       tcp  --  anywhere             anywhere             tcp dpt:61009 to:10.254.0.14:61009
	DNAT       tcp  --  anywhere             anywhere             tcp dpt:61010 to:10.254.0.14:61010
	DNAT       tcp  --  anywhere             anywhere             tcp dpt:61011 to:10.254.0.14:61011
	DNAT       tcp  --  anywhere             anywhere             tcp dpt:61012 to:10.254.0.14:61012

	Chain warden-postrouting (1 references)
	target     prot opt source               destination
	SNAT       all  --  10.254.0.0/22        anywhere             to:10.1.59.242

	Chain warden-prerouting (2 references)
	target     prot opt source               destination
	warden-instance-1762ge8c9k9  all  --  anywhere             anywhere
	
说明：

  * 为61009、61010、61011、61012端口配置了DNAT，即外部访问warden主机61009、61010、61011、61012端的请求的数据包的目标IP地址都会改为10.254.0.14。
  * 为来自10.254.0.0/22地址的数据包配置了SNAT，即从10.254.0.14发出的数据包的源IP地址都会改为warden主机物理网卡eth0的IP地址10.1.59.242。
  * 从实际情况看，61011是warden容器中应用实例使用的端口。其他3个端口没有服务监听。
  * 从dea的相关代码中可以看到有两个端口用于应用实例的console和debug，还有一个端口的用途不明，这部分需要再进一步研究。
	
### 资源分配

  * ip地址：每个容器默认需要4个ip地址。
  * 端口：每个容器默认使用4个端口，范围从61001到65001，总数为4000，因此一台warden主机上可启动的warden容器上限为1000个。
  
### 消息流程总结

  1. 外部对应用实例的访问请求经由router转发到warden主机。
  2. warden主机对请求进行DNAT转换后，将请求转至warden容器进行处理。
  3. warden容器对外部访问的响应和主动对外连接的请求发至warden主机相应的虚拟网卡。
  4. warden主机对消息进行SNAT转换后，向消息发出。

### 遗留问题

  1. 4个端口的使用。
  2. 流量控制。
	
