---
layout: post
category: 云计算
tags: CloudFoundry 容器 虚拟化
description: warden容器的初始化涉及到cgroup、namespaces、网络等linux相关知识，本文对warden容器的初始化过程进行介绍。
---

### 术语
  1. warden主机：运行warden的主机，其上可以运行多个warden容器。
  2. warden容器：运行在warden主机上的虚拟主机，CloudFoundry的每个应用实例就运行在一个warden容器中。
  3. 应用实例：运行在warden容器中的一个应用。

### 创建warden容器
  * 在CloudFoundry中，在应用打包和应用运行都是在warden容器中进行的。在这两种情况下需要，dea通过接口控制warden创建warden容器。
  * 可以通过warden客户端的create命令创建warden容器。
  
### warden容器的创建步骤
  1. 初始化文件系统
  2. 初始化网络
  3. 初始化用户
  4. 初始化用于配置mount的脚本文件
  5. 设置网络
  6. 启动wshd

warden容器创建代码在warden/lib/warden/container/linux.rb中的do_create函数中。

### 初始化文件系统
  1. 将warden/root/linux/skeleton目录复制一份，复制的目录名为容器ID，路径是配置的存放容器的目录。
  2. 在当面容器目录下创建tmp/rootfs、mnt两个目录。
  3. 将tmp/rootfs目录和容器对应的rootfs目录mount到mnt目录下。
  4. 清除mnt/dev下所有的设备文件，根据需要创建相应的设备文件。
  
### 初始化网络
  1. 初始化mnt/etc/hostname文件，写入容器id作为容器的主机名。
  2. 初始化mnt/etc/hosts文件
  3. 把当前主机的resolv.conf文件复制到容器目录相应位置，保证容器启动后，从容器中可以正常访问网络。
  
### 初始化用户
  1. 创建vcap用户
    
### 初始化用于配置mount的脚本文件
  1. 将创建容器请求中的bind_mount参数格式化写入hook-parent-before-clone.sh脚本文件，留待wshd启动时执行。关于wshd启动时执行的四个脚本文件的说明请参考相关部分。
  
### 设置网络
  1. 为容器设置iptable filter规则。
  2. 为容器设置iptable nat规则。
  
关于warden容器网络配置方面的内容请参考[Warden容器的网络配置]

### 启动wshd
  1. 启动wshd进程。wshd进程正常启动后，warden容器就正式创建完成了。

wshd进程是warden容器的核心，关于wshd进程的介绍请参考相关文档。

	
[Warden容器的网络配置]: http://lsword.github.io/2013/09/17.html