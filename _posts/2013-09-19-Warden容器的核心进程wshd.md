---
layout: post
category: 云计算
tags: CloudFoundry 容器 虚拟化
description: 本文对warden容器的核心进程wshd进行介绍。
---

### 术语
  1. warden主机：运行warden的主机，其上可以运行多个warden容器。
  2. warden容器：运行在warden主机上的虚拟主机，CloudFoundry的每个应用实例就运行在一个warden容器中。
  3. 应用实例：运行在warden容器中的一个应用。

### 初始化
  1. 分配一个wshd_t结构，用于与子进程之间传递数据。
  2. 读取参数
    * run_path: server socket文件路径
    * lib_path: hook脚本文件路径
    * root_path: 新根目录路径
  3. 初始化域socket。
  4. 建立用于与子进程之间通信的管道。
  5. 解除与父进程之间对于mount name-space的共享。
  6. 执行hook-parent-before-clone.sh脚本。
  7. clone子进程(CLONE_NEWIPC、CLONE_NEWNET、CLONE_NEWNS、CLONE_NEWPID、CLONE_NEWUTS)。
  8. 将子进程的PID写入环境变量。
  9. 执行hook-parent-after-clone.sh脚本。
    * 配置容器的cgroup
	* 将pid写入pid文件
	* 设置虚拟网卡
  10. 通知clone出的子进程开始运行。
  11. 等待子进程的通知。
  12. 退出。
  
### 服务


	
[Warden容器的网络配置]: http://lsword.github.io/2013/09/17.html