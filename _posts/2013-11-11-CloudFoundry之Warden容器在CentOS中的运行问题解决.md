---
layout: post
category: 云计算
tags: CloudFoundry 容器 虚拟化
description: Warden默认在Ubuntu环境中运行正常，但是如果要是在CentOS中运行，则需要进行一些改动和配置，本文主要介绍如何让Warden在CentOS中正常运行。
---

### 背景
  1. CentOS版本：6.4 64位
  2. warden版本：tag=2fd4fcd33。

### 关闭selinux

CentOS安装后默认开启selinux，warden中使用quota对容器的磁盘空间进行限制，而selinux会对quota有影响，因此需要先关闭selinux。

关闭方法：

	修改/etc/selinux/config文件，将其中的SELINUX=enforcing改为SELINUX=disabled，然后重启系统。

### 配置iptables

CentOS安装后默认的iptables规则会影响warden容器中的程序访问外部网络，从而导致应用打包失败等问题。因此需要对iptables规则配置进行修改。

修改方法：

	修改/etc/sysconfig/iptables文件，修改后内容如下：
	
	# Firewall configuration written by system-config-firewall
	# Manual customization of this file is not recommended.
	*filter
	:INPUT ACCEPT [0:0]
	:FORWARD ACCEPT [0:0]
	:OUTPUT ACCEPT [0:0]
	#-A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
	#-A INPUT -p icmp -j ACCEPT
	#-A INPUT -i lo -j ACCEPT
	#-A INPUT -m state --state NEW -m tcp -p tcp --dport 22 -j ACCEPT
	#-A INPUT -j REJECT --reject-with icmp-host-prohibited
	#-A FORWARD -j REJECT --reject-with icmp-host-prohibited
	COMMIT
	
	修改后执行sudo /etc/init.d/iptables restart重启iptables
	
### 安装相关软件包

	sudo yum -y install glibc-static
	
	
### rootfs

#### 下载rootfs

在warden代码目录执行以下命令

	sudo bundle exec rake --trace setup:rootfs
	
执行此命令后，最终会调用到warden/warden/root/linux/rootfs/centos.sh脚本来进行rootfs的下载。

rootfs的存储位置可以由warden配置文件中server段的container_rootfs_path指定，或者由环境变量CONTAINER_ROOTFS_PATH指定。默认位置是/tmp/warden/rootfs。

#### 在rootfs中创建相应目录和文件

	sudo chroot /tmp/warden/rootfs
	mkdir /app
	mkdir /build_cache
	cd etc
	touch fstab
	touch resolv.conf
	echo 'nameserver x.x.x.x' > resolv.conf
	mknod -m 644 /dev/urandom c 1 9
	
#### 在rootfs中安装必要的软件包

	sudo chroot /tmp/warden/rootfs
	yum update
	yum -y install zip
	yum -y install wget
	yum -y install curl
	yum -y install git
	

#### 在rootfs中安装ruby

	sudo chroot /tmp/warden/rootfs
	curl http://pyyaml.org/download/libyaml/yaml-0.1.4.tar.gz -s -o yaml-0.1.4.tar.gz
	tar xfvz yaml-0.1.4.tar.gz
	rm yaml-0.1.4.tar.gz
	cd yaml-0.1.4
	./configure --prefix=/yaml
	make;make install
	curl http://cache.ruby-lang.org/pub/ruby/ruby-1.9.3-p429.tar.gz -s -o ruby-1.9.3-p429.tar.gz
	tar xfvz ruby-1.9.3-p429.tar.gz
	rm ruby-1.9.3-p429.tar.gz
	cd ruby-1.9.3-p429
	./configure --enable-load-relative --disable-install-doc --with-opt-dir=/yaml
	make
	make install
	ln -s /usr/local/bin/ruby /usr/bin/ruby
	
#### 修改脚本文件common.sh

文件位置：warden/warden/root/linux/skeleton/lib/common.sh

将setup_fs_other函数修改成以下内容：

	function setup_fs_other() {
	  mkdir -p tmp/rootfs mnt mnt/app
	  mkdir -p $rootfs_path/proc

	  mount -n --bind $rootfs_path mnt
	  mount -n --bind -o remount,ro $rootfs_path mnt

	  overlay_directory_in_rootfs /dev rw
	  overlay_directory_in_rootfs /etc rw
	  overlay_directory_in_rootfs /home rw
	  overlay_directory_in_rootfs /sbin rw
	  overlay_directory_in_rootfs /buildpack_cache rw
	  overlay_directory_in_rootfs /app rw

	  mkdir -p tmp/rootfs/tmp
	  chmod 1777 tmp/rootfs/tmp
	  overlay_directory_in_rootfs /tmp rw
	}
	
### 总结

未完待续...
	
[Linux中的namespaces]: http://lsword.github.io/2013/09/20.html
[sched-bwc]: https://www.kernel.org/doc/Documentation/scheduler/sched-bwc.txt
[sched-rt-group]: https://www.kernel.org/doc/Documentation/scheduler/sched-rt-group.txt