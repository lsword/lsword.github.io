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


	
[Linux中的namespaces]: http://lsword.github.io/2013/09/20.html