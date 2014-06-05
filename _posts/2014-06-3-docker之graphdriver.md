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

#### 3.3 文件说明

docker使用devicemapper驱动时，/var/lib/docker目录下会存在一个devicemapper目录，包含devicemapper和mnt两个子目录。

##### 3.3.1 devicemapper

此目录下有三个文件，data、json、metadata。

其中data、metadata就是dmsetup用到的数据文件。data容量上限为100G、metadata容量上限为2G。所有container的空间都是在data文件中分配的，container删除时会从data中回收分配的空间（测试方法：可以启动一个container，执行dd if=/dev/zero of=test bs=1M count=100命令，在container创建一个100M的文件，然后使用du命令观察data文件大小的变化。最后退出并删除container，观察data文件大小的变化）。

---

json文件保存data文件中的设备信息，我们可以通过查看各个阶段的json文件内容来进一步了解json文件的作用。

docker初次启动时，json文件内容如下：

~~~
{"Devices":{
	"":{"device_id":0,"size":10737418240,"transaction_id":1,"initialized":true}}}
~~~

这是因为在初始化时，需要创建一个BaseImage。

---

使用docker pull命令获取ubuntu/12.04镜像后，json内容如下：

~~~
{"Devices":{
	"":{"device_id":0,"size":10737418240,"transaction_id":1,"initialized":true},
	"511136ea3c5a64f264b78b5433614aec563103b4d4702f3ba7d4d2698e22c158":{"device_id":1,"size":10737418240,"transaction_id":2,"initialized":true},
	"5dbd9cb5a02fb27734e3dbaef8d6abf0997c137f49dd433bf3f27c8036d3348e":{"device_id":4,"size":10737418240,"transaction_id":5,"initialized":true},
	"74fe38d114018aac73c5997b95263090048ec9a1f58f33a1b53f55e92156d53b":{"device_id":5,"size":10737418240,"transaction_id":6,"initialized":true},
	"82cdea7ab5b555f53c2adf8df75b0d2ad1e49dbfc11da50df3e7ea38454ed606":{"device_id":3,"size":10737418240,"transaction_id":4,"initialized":true},
	"d0ca5d7c3fc048a9c7363d52315d829959078b4e153ef30bdf79ac6ee44c12b3":{"device_id":6,"size":10737418240,"transaction_id":7,"initialized":true},
	"f10ebce2c0e158af1eb0dc08c9e917cc0976e7d57319defbb06ea61191d29e76":{"device_id":2,"size":10737418240,"transaction_id":3,"initialized":true}}
}
~~~

为各层Image都创建了一个device。

---

使用docker run命令运行一个容器后，json内容如下：

~~~
{"Devices":{
	"":{"device_id":0,"size":10737418240,"transaction_id":1,"initialized":true},
	"511136ea3c5a64f264b78b5433614aec563103b4d4702f3ba7d4d2698e22c158":{"device_id":1,"size":10737418240,"transaction_id":2,"initialized":true},
	"5dbd9cb5a02fb27734e3dbaef8d6abf0997c137f49dd433bf3f27c8036d3348e":{"device_id":4,"size":10737418240,"transaction_id":5,"initialized":true},
	"6170362ca522d16c6f286c9cdde59a815006694c59bc578342785ccd3fd01161":{"device_id":16,"size":10737418240,"transaction_id":25,"initialized":true},
	"6170362ca522d16c6f286c9cdde59a815006694c59bc578342785ccd3fd01161-init":{"device_id":15,"size":10737418240,"transaction_id":24,"initialized":true},
	"74fe38d114018aac73c5997b95263090048ec9a1f58f33a1b53f55e92156d53b":{"device_id":5,"size":10737418240,"transaction_id":6,"initialized":true},
	"82cdea7ab5b555f53c2adf8df75b0d2ad1e49dbfc11da50df3e7ea38454ed606":{"device_id":3,"size":10737418240,"transaction_id":4,"initialized":true},
	"d0ca5d7c3fc048a9c7363d52315d829959078b4e153ef30bdf79ac6ee44c12b3":{"device_id":6,"size":10737418240,"transaction_id":7,"initialized":true},
	"f10ebce2c0e158af1eb0dc08c9e917cc0976e7d57319defbb06ea61191d29e76":{"device_id":2,"size":10737418240,"transaction_id":3,"initialized":true}}
}
~~~

可以看到多出了6170362ca522d16c6f286c9cdde59a815006694c59bc578342785ccd3fd01161和6170362ca522d16c6f286c9cdde59a815006694c59bc578342785ccd3fd01161-init两个设备。

---

删除刚才创建的容器后，容器id相关的两个设备信息在json文件中消失。

##### 3.3.2 mnt

docker初始化后，mnt目录为空。

使用docker pull命令获取image后，会在mnt目录中为每级image创建一个空的子目录。

启动一个容器后，会在mnt目录中创建两个目录，容器id目录和容器id-init目录。其中容器id-init目录为空。

##### 3.4 mount信息

对于同一个容器来说，以下两个目录都是mount到同一个设备上的。

	/var/lib/docker/devicemapper/mnt/容器id
	/var/lib/docker/containers/容器id/root

通过系统的mount命令或df命令是无法查看的，可以通过/proc/docker进程id下的mountinfo、mounts、mountstats三个文件查看设备的mount情况。

#### 3.5 存在的问题

##### 3.5.1 容器相关设备清除问题。

在删除容器时，如果正在访问/var/lib/docker/devicemapper/mnt/容器id/rootfs或者/var/lib/docker/container/容器id/rootfs，则无法删除容器。此时，即使重启docker服务也无法删除容器，目前看来，只有重启主机才能再删除容器。

##### 3.5.2 空间限制

data最大容量100G，单个容器最大容量10G，这个是写死在代码中的。虽然一般情况下不会达到上限，但是毕竟是个限制。

### 4、aufs

aufs是docker在支持aufs文件系统的Linux上运行时使用的驱动程序。实现要比devmapper简单的多。

#### 4.1、初始化

通过查看/proc/filesystem文件确定是否支持aufs。

在/var/lib/docker/aufs目录下创建mnt、diff、layers三个目录。

#### 4.2 文件说明

##### 4.2.1 mnt

与devmapper的mnt作用一致。

##### 4.2.2 layers

容器或镜像的继承层次。

##### 4.2.3 diff

容器或镜像的每一层与其父层不同的文件。

#### 4.3 关键函数

##### 4.3.1 Create

在/var/lib/docker/aufs目录中的mnt和diff子目录中为容器建立相应的子目录。

在/var/lib/docker/aufs目录中的layers目录中为容器建立相应的layer文件。

##### 4.3.2 Get

调用aufsmount函数为容器挂载文件系统。

### 5、btrfs

待补充

### 6、vfs

待补充

### 参考文档

[device-mapper]
[thinprovisioned_volumes]
[thin-provisioning.txt]



[thinprovisioned_volumes]: https://access.redhat.com/site/documentation/en-US/Red_Hat_Enterprise_Linux/6/html/Logical_Volume_Manager_Administration/thinprovisioned_volumes.html
[device-mapper]: http://device-mapper.com/
[thin-provisioning.txt]:https://github.com/torvalds/linux/blob/master/Documentation/device-mapper/thin-provisioning.txt