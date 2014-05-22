---
layout: post
category: 云计算 虚拟化
tags: docker
description: 本文介绍docker容器互联工具pipework
---

### 背景

本文中涉及的源码、测试环境都是基于ubuntu server 14.04下的docker 0.11.1版本系统。

pipework（[代码地址]、[docker官方介绍]）是解决docker容器互联的一个工具脚本，只有不到200行代码。基本思路是把docker挂载到主机上的另外一个网桥（非docker自用的网桥）或者物理网卡设备上并指定一个新的IP地址。如果两台主机之间的网桥或者物理网卡设备可以互通，那么两台主机上的docker容器就可以相互访问。

pipework代码功能不复杂，但是通过分析代码，可以了解到网络虚拟化的一些细节。

### 使用说明

~~~
为容器手动分配ip
pipework <hostinterface> [-i containerinterface] <guest> <ipaddr>/<subnet>[@default_gateway] [macaddr]

使用dhcp为容器分配ip
pipework <hostinterface> [-i containerinterface] <guest> dhcp [macaddr]
~~~

hostinterface为主机的网络接口设备名称，可以是网桥或物理网卡。

containerinterface为容器的网络接口设备名称，默认为eth1。

guest为容器id。

### 代码实现

~~~
1、判断hostinterface是网桥设备还是物理网卡。如果是网桥设备，还需要区分是linux自带网桥还是openvswitch。
2、根据guest找到相应docker容器。
3、处理ipaddr，如果是dhcp则需要安装udpcpc，如果是ip地址则对有效性进行判断。
4、创建一个netns，名称为docker容器的进程。
5、判断是否需要创建网桥，如果hostinterface的命名以br开头，但是/sys/class/net/<hostinterface>/bridge目录又不存在的话，则创建一个网桥设备。
6、如果hostinterface是网桥设备则创建一个veth对，并且把主机侧的veth挂到网桥上。
7、如果hostinterface是物理网卡的话，则讲解一个macvlan设备。
8、把6或7步中创建的虚拟设备加入容器的网络空间，并设置设备名和mac地址（如果指定的话）。
9、为容器中的veth设备或macvlan设备设置ip地址、gateway地址。
~~~

### 存在的问题

使用docker stop命令停止容器后，pipework在/var/run/netns中为容器创建的链接未删除。

### 学习总结

~~~
1、/sys/class/net目录中保存着到网络设备目录的链接。
2、虚拟网络设备文件保存在/sys/dev/virtual/net目录。
3、网桥设备的目录中有一个bridge目录，可以以此来判断一个网络设备是否为网桥。
4、/var/run/netns目录中保存网络命名空间相关文件，每个网络命名空间在都有一个目录，目录名即为网络命名空间名。
5、如果要查看一个进程的网络命名空间，可以在/var/run/netns中建立一个到/proc/进程号/ns/net的链接，然后即可通过ip netns exec命令查看。
~~~

[代码地址]: https://github.com/jpetazzo/pipework
[docker官方介绍]: http://docs.docker.io/use/networking/