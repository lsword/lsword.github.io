---
layout: post
category: 云计算
tags: 容器 虚拟化
description: 本文对Linux操作系统中的namespaces技术进行介绍。
---

### 概述
  Linux中的namespaces技术是在Linux平台实现容器这种轻量级虚拟化的基础，这是Linux内容提供的支持，lxc、warden等容器都用到了Linux中的namespaces。

### CLONE_NEWNS（Linux 2.4.19）

  
  ~~~
  
	CLONE_NEWNS对应Mount namespaces。
  
	Mount namespaces为一组进程隔离了一系列的文件系统挂载点。在不同Mount namespaces的进程有不同的文件系统层次视图。通过使用Mount namespaces，进程调用mount()和umount()函数时不再操作对所有进程可见的全局挂载点，而是只操作调用进程所关联的mount namespace。
	
	使用Mount namespaces可以象chroot方式一样创建环境。但是，与使用chroot调用对比，Mount namespaces方式更加安全和灵活。
	
	Mount namespaces是Linux中实现的第一种namespace（2002年）。
	
  ~~~
  
### CLONE_NEWUTS（Linux 2.6.19）
  
  ~~~
  
	CLONE_NEWUTS对应UTS namespaces。
  
	UTS namespaces实现uname()系统调用返回的对两个系统描述符nodename和domainname的隔离（这两个值可以由sethostname()和setdomainname()系统调用来设置）。
	
	在容器的上下文中，UTS namespaces允许每个容器有自己的hostname和NIS domain name。这对于基于这两个名字的初始化和配置脚本很有用。

  ~~~

### CLONE_NEWIPC（Linux 2.6.19）
  
  ~~~
   
	CLONE_NEWIPC对应IPC namespaces。
  
	IPC namespaces实现进程间通信(IPC)资源、System V IPC对象、POSIX消息队列的隔离。每个IPC namespace有自己的System V IPC描述符和POSIX消息队列。

  ~~~

### CLONE_NEWPID（Linux 2.6.24）
  
  ~~~
   
	CLONE_NEWPID对应PID namespaces。
  
	PID namespaces实现进程间ID空间隔离。在不同PID namespaces中的进程可以有相同的PID。PID namespaces的一个主要好处是容器可以在不同的主机上移植，保持容器内部的进程ID不变。
	
	PID namespaces还允许每个容器有自己的init进程，及PID为1的进程，这个进程是容器中所有进程的祖先。

	一个warden容器的wshd进程就有两个pid，一个是容器内部的PID（1），另外一个是容器外部的PID。
	
	PID namespaces是可以嵌套。
	
	一个进程只能看到其所在的PID namespace和其所嵌套的下级PID namespace中的进程。
  ~~~

### CLONE_NEWNET（Linux 2.6.24 - Linux 2.6.29）
  
  ~~~
   
	CLONE_NEWNET对应Network namespaces。
  
	NET namespaces提供对与网络相关的系统资源的隔离。每个Network namespace有自己的网络设备、IP地址、IP路由表、/proc/net目录，端口号等等。
	
	同一台主机上的多个容器中的web服务器都可以绑定80端口。	
  ~~~

### CLONE_NEWUSER（Linux 2.6.23 - Linux 3.8）
  
  ~~~
   
	CLONE_NEWUSER对应User namespaces。
  
	User namespaces提供对用户id和组id号码空间的资源隔离。在一个user namespaces内部和外部，一个进程有不同的userid和groupid。
	
	User namespace可以让一个进程在User namespace内有root权限，而在User namespace外则只有普通权限。

  ~~~


