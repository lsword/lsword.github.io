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

#### 父进程(wshd->parent_run)
  1. 分配一个wshd_t结构，用于与子进程之间传递数据。
  2. 读取参数
    * run_path: server socket文件路径
    * lib_path: hook脚本文件路径
    * root_path: 新根目录路径
  3. 初始化域socket，用于与wsh进程通信。
  4. 建立用于与子进程之间通信的管道。

  ~~~
	barrier_parent：用于父进程通知子进程开始运行。
	barrier_child：用于子进程通知父进程自己即将进入主循环，准备开始正式提供服务。父进程收到子进程的通知后退出。
  ~~~

  5. 解除与父进程之间对于mount name-space的共享。
  6. 执行hook-parent-before-clone.sh脚本。
  
  ~~~
	此脚本主要工作是mount上容器运行时需要的一些目录。这些内容是在warden/lib/warden/container/linux.rb的do_create函数中调用write_bind_mount_commands动态生成的。
  ~~~
  
  7. clone子进程(CLONE_NEWIPC、CLONE_NEWNET、CLONE_NEWNS、CLONE_NEWPID、CLONE_NEWUTS)。
  
  ~~~
  
  ~~~
  
  8. 将子进程的PID写入环境变量。
  9. 执行hook-parent-after-clone.sh脚本。
    * 配置容器的cgroup
	* 将pid写入pid文件
	* 设置虚拟网卡
  10. 通知clone出的子进程开始运行。
  
  ~~~
  	barrier_signal(&w->barrier_parent);
  ~~~
  
  11. 等待子进程的通知。
  
  ~~~
  	barrier_wait(&w->barrier_child);
  ~~~
  
  12. 退出。
  
#### 子进程(wshd->child_run)
  1. 等待父进程的通知，接收到父进程通知后继续执行。
  2. 执行hook-child-before-pivot.sh脚本。
  
  ~~~
	  目前没有实际功能。
  ~~~
  
  3. pivot_root。
  4. 执行hook-child-after-pivot.sh脚本。
  
  ~~~
  	准备伪终端
  	准备proc
  	配置网卡和路由
  ~~~
  
  5. execl("/sbin/wshd", "/sbin/wshd", "--continue", NULL);
  6. child_load_from_shm
  
  ~~~
  	读取父进程放在共享内存中的wshd_t结构。
  ~~~
  
  7. mount_umount_pivoted_root
  8. setsid
  
  ~~~
  	退出原来的进程组，成为一个daemon。
  ~~~
  
  9. 通知父进程退出。
  
  ~~~
	barrier_signal(&w->barrier_child);
  ~~~
  
  10. 进入主循环，等待wsh发出的请求。
  
  ~~~
  	child_loop(w);
  ~~~

### 服务


	
[Warden容器的网络配置]: http://lsword.github.io/2013/09/17.html