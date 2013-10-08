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

### 关键数据结构

#### wshd_t
	struct wshd_s {
		/* Path to directory where server socket is placed */
		char run_path[256]; //server socket的保存路径

		/* Path to directory containing hooks */
		char lib_path[256]; //钩子脚本的路径

		/* Path to directory that will become root in the new mount namespace */
		char root_path[256]; //新的挂载命名空间的根目录路径

		/* Process title */
		char title[32]; //进程标题

		/* File descriptor of listening socket */
		int fd; //监听的域socket的文件描述符

		barrier_t barrier_parent;
		barrier_t barrier_child;

		/* Map pids to exit status fds */
		//保存每个子进程的pid和用于与发出创建子进程请求的客户端进行通信的管道写端文件描述符。
		struct {
			pid_t pid;
			int fd;
		} *pid_to_fd;
		size_t pid_to_fd_len;
	};


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
	/* Point to top of stack (it grows down) */
	stack = stack + pagesize;

	/* Setup namespaces */
	flags |= CLONE_NEWIPC;
	flags |= CLONE_NEWNET;
	flags |= CLONE_NEWNS;
	flags |= CLONE_NEWPID;
	flags |= CLONE_NEWUTS;

	pid = clone(child_run, stack, flags, w);  
  ~~~
  
  此处涉及到Linux中的namespaces技术，请参考[Linux中的namespaces]

  8. 将子进程的PID写入环境变量。
  9. 执行hook-parent-after-clone.sh脚本。
  
    9.1 配置容器的cgroup
    
      	
		for system_path in /tmp/warden/cgroup/*
		do
		  instance_path=$system_path/instance-$id

		  mkdir -p $instance_path

		  if [ $(basename $system_path) == "cpuset" ]
		  then
		    cat $system_path/cpuset.cpus > $instance_path/cpuset.cpus
		    cat $system_path/cpuset.mems > $instance_path/cpuset.mems
		  fi

		  if [ $(basename $system_path) == "devices" ]
		  then
		    # disallow everything, allow explicitly
		    echo a > $instance_path/devices.deny
 		   # /dev/null
		    echo "c 1:3 rw" > $instance_path/devices.allow
		    # /dev/zero
		    echo "c 1:5 rw" > $instance_path/devices.allow
		    # /dev/random
		    echo "c 1:8 rw" > $instance_path/devices.allow
		    # /dev/urandom
		    echo "c 1:9 rw" > $instance_path/devices.allow
		    # /dev/tty
		    echo "c 5:0 rw" > $instance_path/devices.allow
		    # /dev/ptmx
		    echo "c 5:2 rw" > $instance_path/devices.allow
		    # /dev/pts/*
		    echo "c 136:* rw" > $instance_path/devices.allow
		  fi
		    echo $PID > $instance_path/tasks
		done
		
        可以看到，此时只设置cgroup中的cpuset和devices。其他部分设置是容器初始化成功后，dea控制warden进行设置的。


    9.2 将pid写入pid文件

		echo $PID > ./run/wshd.pid

    9.3 设置虚拟网络
		
		ip link add name $network_host_iface type veth peer name 		$network_container_iface
		ip link set $network_host_iface netns 1
		ip link set $network_container_iface netns $PID

		ifconfig $network_host_iface $network_host_ip netmask $network_netmask
		

		* 建立一对veth设备，名字分别是 $network_host_iface 和 $network_container_iface。这两个设备是完全对称的，从其中一个发出消息就会从另一个收到。（veth的作用就是要把从一个network namespace发出的数据包转发到另一个network namespace。veth 设备是成对的，一个是container中，另一个在真实机器上。）
		* 把设备$network_host_iface放到主机的网络命名空间中。
		* 把设备$network_container_iface放到容器的网络命名空间中。
		* 设置$network_host_iface设备的IP地址。


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
  
		目前没有实际功能。

  3. pivot_root(".", "mnt");
  
		把当前进程的文件系统目录移到容器目录中的mnt目录，把当前进程的根文件系统设置为容器目录。移除对之前根文件系统的依赖。

  4. 执行hook-child-after-pivot.sh脚本。

    4.1 准备伪终端

		mkdir -p /dev/pts
		mount -t devpts -o newinstance,ptmxmode=0666 devpts /dev/pts
		ln -sf pts/ptmx /dev/ptmx

    4.2 准备proc

		mkdir -p /proc
		mount -t proc none /proc

    4.3 设置主机名

		hostname $id

    4.4 配置网卡和路由

		ifconfig lo 127.0.0.1
		ifconfig $network_container_iface $network_container_ip netmask $network_netmask mtu $container_iface_mtu
		route add default gw $network_host_ip $network_container_iface

		* 设置容器中的环回设备。
		* 设置容器中$network_container_iface设备的IP地址。
		* 配置容器路由。
		
  5. execl("/sbin/wshd", "/sbin/wshd", "--continue", NULL);

    5.1 child_load_from_shm
  
		读取父进程放在共享内存中的wshd_t结构。

    5.2 mount_umount_pivoted_root

		umount原来的根文件系统。

    5.3 setsid
  
		退出原来的进程组，成为一个daemon。

    5.4 通知父进程退出。
  
		barrier_signal(&w->barrier_child);

    5.5 进入主循环，等待wsh发出的请求。
  
		child_loop(w);

### 服务

wshd进程初始化完成后，进入到child_loop函数。这个函数主要处理两方面工作：

#### 等待处理外部的请求
  
    等待处理外部的请求分为两种：

	* 交互式：用户通过可执行程序wsh与wshd进行交互。
	* 非交互式：DEA通过向warden服务器发送指令与wshd进行交互。
	
    对于这两种请求方式，wshd分别使用child_handle_interactive和child_handle_noninteractive函数进行处理。这两个函数都会调用child_fork函数来处理请求。

	
##### child_fork(请求消息, in, out, err)

这个函数的主要目的就是fork出子进程，执行请求消息中的命令。

  1. fork出子进程。
  2. 通过dup2函数，将子进程的标准输入、标准输出、标准错误输出分别设置为传入参数中的in、out、err文件描述符。
  3. 调用setsid使子进程在一个单独的会话中运行。
  4. 设置登陆用户，默认为root。
  5. 使用getpwnam函数获取用户的登陆相关信息（密码、用户id、组id、真实名称、home目录、shell程序等）。
  6. 如果输入来自控制终端，则通过ioctl打开控制终端。
  7. 从请求消息中读取命令参数。
  8. 从请求消息中读取rlimit信息并进行设置。
  9. 从请求消息中读取用户用户的gid和uid并进行设置。
  10. 设置环境变量。
  11. 使用execvpe函数执行请求指令。


##### 交互式

  1. 初始化一个管道用于进程间通信。
  2. 初始化一个伪终端。
  3. 将管道的读端文件描述符和伪终端的master文件描述符发给请求方（wsh进程）。
  4. child_fork(请求消息，伪终端的slave文件描述符，伪终端的slave文件描述符，伪终端的slave文件描述符)
  5. 将child_fork返回的pid和用于通信的管道写端文件描述符记入wshd_t结构中。
  

##### 非交互式

  1. 初始化4个管道。管道1用于标准输入、管道2用于标准输出、管道3用于标准错误输出、管道4用于进程间通信。
  2. 将管道1的写描述符、管道2的读描述符、管道3的读描述符、管道4的读描述符发送给请求方（wsh进程）。
  3. child_fork(请求消息，管道1的读描述符，管道2的写描述符，管道3的写描述符)
  4. 将child_fork返回的pid和用于通信的管道(管道4)写端文件描述符记入wshd_t结构中。
	
#### 等待处理子进程的信号

wshd进入主循环后，调用child_signalfd函数来设置处理子进程的SIGCHLD信号。

如果接收到子进程的SIGCHLD信号，则调用child_handle_sigchld函数来进行响应处理。

##### child_signalfd

  1. 清空信号集。
  2. 设置接收子进程的SIGCHLD信号。
  3. 通过signalfd系统调用返回一个文件描述符，外部函数可以通过检测这个文件描述符来判断是否有收到子进程退出信号。
  
  
##### child_handle_sigchld

  1. 调用waitpid函数等待子进程结束并获取子进程退出信息。
  2. 删除wshd_t结构中的pid和fd信息。
  3. 如果子进程正常结束，则将子进程的退出信息通过管道发送给客户端(wsh)。
  4. 如果子进程非正常结束，则不对客户端(wsh)进行通知(?)。
  5. 关闭与客户端(wsh)之间的管道。

	
[Linux中的namespaces]: http://lsword.github.io/2013/09/20.html