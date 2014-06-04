---
layout: post
category: 云计算 虚拟化
tags: docker
description: 本文介绍docker容器中实现容器文件系统管理的graphdriver
---

### 1、背景

本文中涉及的源码基于docker 0.11.1版本系统。
本文中涉及的测试环境基于ubuntu server 14.04及redhat6.5。

docker对于容器文件系统的管理依赖于宿主机文件系统的支持，最初docker只能在支持aufs文件系统的Linux上运行。而实际上各个Linux发行版中支持的文件系统类型不一致，例如目前的Redhat6.5版本中就没有提供对aufs的支持。docker中为了实现好的兼容性、扩展性，通过graphdriver这种可扩展的方式来实现对不同文件系统的支持。

docker目前支持aufs、btrfs、devicemapper、vfs四种文件系统。

### 2、概述

graphdriver源码位于docker/daemon目录下。

driver.go文件中定义了Driver的接口，包括String、Create、Remove、Get、Put、Exists、Status、Cleanup等函数。

aufs、btrfs、devmapper、vfs等四个目录分别是对四种不同文件系统的支持。

### 3、devmapper

devmapper是docker在redhat6.5上运行时使用的驱动程序。使用到了Linux系统的loops设备、device mapper技术。

#### 3.1、初始化过程

~~~
在/var/lib/docker/devicemapper/devicemapper目录下创建两个稀疏文件，data(100G)和metadata(2G)。
将data和metadata分别挂载到/dev/loop0和/dev/loop1上（查找/dev目录中可用的loop设备，如果没有其他程序使用，就会用/dev/loop0和/dev/loop1）。
创建device mapper的thin-pool，data文件对应thin-pool的target device、metadata文件对应thin-pool的mapped device。
	dmsetup create docker-253:0-655790-pool --table "0 209715200 thin-pool /dev/loop1 /dev/loop0 128 32768 1 skip_block_zeroing"
初始化metadata。
创建BaseImage。
	创建设备（相当于执行以下命令）
		dmsetup message docker-253:0-655790-pool 0 "create_thin 3"
		对应的删除命令：dmsetup message docker-253:0-655790-pool 0 "delete 3"
	注册设备
		把设备信息写入/var/lib/docker/devicemapper/devicemapper/json文件。
	激活设备（相当于执行以下命令）
		dmsetup create docker-253:0-655790-base --table "0 20971520 thin /dev/mapper/docker-253:0-655790-pool 3"
		对应的删除命令：dmsetup remove docker-253:0-655790-base
	创建文件系统
		mkfs.ext4 -E discard,lazy_itable_init=0 /dev/mapper/docker-253:0-655790-base
		此时会在/var/lib/docker/devicemapper/devicemapper/data文件中分配290多M的空间。
	保存metadata
~~~

初始化结束后，系统情况如下：

~~~
使用dmsetup info命令能看到以下信息。
	Name:              docker-253:0-655790-pool
	State:             ACTIVE
	Read Ahead:        256
	Tables present:    LIVE
	Open count:        1
	Event number:      0
	Major, minor:      253, 2
	Number of targets: 1
	
	Name:              docker-253:0-655790-base
	State:             ACTIVE
	Read Ahead:        256
	Tables present:    LIVE
	Open count:        0
	Event number:      0
	Major, minor:      253, 3
	Number of targets: 1
		
使用fdisk -l命令能看到新的磁盘。
	Disk /dev/mapper/docker-253:0-655790-pool: 107.4 GB, 107374182400 bytes
	255 heads, 63 sectors/track, 13054 cylinders
	Units = cylinders of 16065 * 512 = 8225280 bytes
	Sector size (logical/physical): 512 bytes / 512 bytes
	I/O size (minimum/optimal): 512 bytes / 65536 bytes
	Disk identifier: 0x00000000
		
	Disk /dev/mapper/docker-253:0-655790-base: 10.7 GB, 10737418240 bytes
	255 heads, 63 sectors/track, 1305 cylinders
	Units = cylinders of 16065 * 512 = 8225280 bytes
	Sector size (logical/physical): 512 bytes / 512 bytes
	I/O size (minimum/optimal): 512 bytes / 65536 bytes
	Disk identifier: 0x00000000
~~~

#### 3.2、创建rootfs

docker创建容器前需要先为容器准备rootfs，此时会依次调用graphdriver的Create和Get函数。对应到devmapper中就是DeviceSet.AddDevice和DeviceSet.MountDevice函数

DeviceSet.AddDevice

~~~
创建快照设备
	dmsetup message docker-253:0-655790-pool 0 "create_snap 设备id base设备id"
注册设备
	更新json文件
~~~

DeviceSet.MountDevice

~~~
激活设备
	dmsetup create 设备名称 --table "0 20971520 thin /dev/mapper/docker-253:0-655790-pool 设备号"
挂载设备
	mount /dev/mapper/设备名称 /var/lib/docker/containers/容器id/rootfs
~~~

### 参考文档

[device-mapper]
[thinprovisioned_volumes]
[thin-provisioning.txt]



[thinprovisioned_volumes]: https://access.redhat.com/site/documentation/en-US/Red_Hat_Enterprise_Linux/6/html/Logical_Volume_Manager_Administration/thinprovisioned_volumes.html
[device-mapper]: http://device-mapper.com/
[thin-provisioning.txt]:https://github.com/torvalds/linux/blob/master/Documentation/device-mapper/thin-provisioning.txt