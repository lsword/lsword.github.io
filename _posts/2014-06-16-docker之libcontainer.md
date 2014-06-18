---
layout: post
category: 云计算 虚拟化
tags: docker
description: 本文介绍docker使用的容器管理工具库libcontainer
---

### 1 背景

本文中涉及的源码基于docker 1.0版本系统。
本文中涉及的测试环境基于ubuntu server 14.04及redhat6.5。

### 2 概述

启动docker服务时可以指定docker的容器运行驱动，默认情况下使用native方式运行容器（请参考[Docker之execdriver]）。在这种方式下，是使用libcontainer库来实现基于操作系统的轻量级虚拟化的。

本文首先介绍libcontainer中自带的nsinit工具程序的使用，先对其有一个直观的认识，然后再介绍libcontainer的实现。

### 3 nsinit

nsinit是libcontainer中自带的一个基于libcontainer的工具程序。使用这个工具可以创建容器、进入到一个已有的容器中等等。是一个非常有用的工具。

以下为nsinit的功能参数列表。

~~~
exec		execute a new command inside a container
init		runs the init process inside the namespace
stats		display statistics for the container
spec		display the container specification
nsenter		init process for entering an existing namespace
~~~

#### 3.1 安装

以下为nsinit的安装过程。

~~~
paas@ubuntu:~$ pwd
/home/paas
paas@ubuntu:~$ mkdir -p libcontainer/src
paas@ubuntu:~$ cd libcontainer/src
paas@ubuntu:~/libcontainer/src$ git clone https://github.com/docker/libcontainer.git
Cloning into 'libcontainer'...
remote: Reusing existing pack: 1721, done.
remote: Counting objects: 38, done.
remote: Compressing objects: 100% (37/37), done.
remote: Total 1759 (delta 12), reused 0 (delta 0)
Receiving objects: 100% (1759/1759), 354.50 KiB | 72.00 KiB/s, done.
Resolving deltas: 100% (1111/1111), done.
Checking connectivity... done.
paas@ubuntu:~/libcontainer/src$ ls
libcontainer
paas@ubuntu:~/libcontainer/src$ cd /home/paas/libcontainer
paas@ubuntu:~/libcontainer$ GOPATH=/home/paas/libcontainer
paas@ubuntu:~/libcontainer$ go get -v ./...
github.com/docker/libcontainer (download)
github.com/dotcloud/docker (download)
github.com/syndtr/gocapability (download)
github.com/coreos/go-systemd (download)
github.com/godbus/dbus (download)
github.com/codegangsta/cli (download)
package github.com/coreos/go-systemd/activation
	imports github.com/coreos/go-systemd/dbus
	imports github.com/godbus/dbus
	imports code.google.com/p/go.net/websocket
github.com/gorilla/mux (download)
github.com/gorilla/context (download)
package github.com/coreos/go-systemd/activation
	imports github.com/coreos/go-systemd/dbus
	imports github.com/godbus/dbus
	imports github.com/gorilla/mux
	imports github.com/gorilla/context
	imports code.google.com/p/gosqlite/sqlite3
github.com/kr/pty (download)
package github.com/coreos/go-systemd/activation
	imports github.com/coreos/go-systemd/dbus
	imports github.com/godbus/dbus
	imports github.com/gorilla/mux
	imports github.com/gorilla/context
	imports github.com/kr/pty
	imports code.google.com/p/go.net/html/atom
paas@ubuntu:~/libcontainer$ go build github.com/docker/libcontainer/nsinit
paas@ubuntu:~/libcontainer$ ls
nsinit  pkg  src
~~~

安装结束后，会出现一个nsinit的可执行文件。

#### 3.2 nsinit exec

在已有的容器中运行一个命令。

> 为了测试这个功能，我们需要先启动一个容器。

~~~
paas@ubuntu:~/libcontainer$ docker run -d -p 6379:6379 shipyard/redis
fa6f79b1683273bb56257c57b7961f606b3f37be40ad14dc59d1ef394999a72d
~~~

> 在这个容器中执行一个ps aux命令来查看容器中的进程情况。

~~~
root@ubuntu:/# cd /var/lib/docker/execdriver/native/fa6f79b1683273bb56257c57b7961f606b3f37be40ad14dc59d1ef394999a72d
root@ubuntu:/# root@ubuntu:/var/lib/docker/execdriver/native/fa6f79b1683273bb56257c57b7961f606b3f37be40ad14dc59d1ef394999a72d# /home/paas/libcontainer/nsinit exec ps aux
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.0  0.7  35012  7600 ?        Ssl  02:49   0:00 /usr/local/bin/redis-server /etc/redis.conf
root        10  0.0  0.1  15276  1128 ?        R+   02:53   0:00 ps aux
~~~

可以看到命令的结果显示的是容器中的进程情况。

> 在这个容器中执行一个shell。

~~~
root@ubuntu:/var/lib/docker/execdriver/native/fa6f79b1683273bb56257c57b7961f606b3f37be40ad14dc59d1ef394999a72d# /home/paas/libcontainer/nsinit exec bash
root@fa6f79b16832:/# ps -aux
Warning: bad ps syntax, perhaps a bogus '-'? See http://procps.sf.net/faq.html
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.0  0.7  35012  7600 ?        Ssl  02:49   0:00 /usr/local/bin/redis-server /etc/redis.conf
root        20  0.0  0.1  18024  1848 ?        S    02:54   0:00 bash
root        27  0.0  0.1  15276  1132 ?        R+   02:56   0:00 ps -aux
root@fa6f79b16832:/#
~~~

这时bash程序运行在容器中，我们得到了一个容器中的shell，可以在容器中进行各种操作。

此时nsinit程序仍在运行，可以观察一下nsinit进程的情况。

~~~
root@ubuntu:/# ps -ef | grep nsinit
root     26825 26790  0 10:54 pts/0    00:00:00 /home/paas/libcontainer/nsinit nsenter 26742  {"hostname":"fa6f79b16832","environment":["HOME=/","PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin","HOSTNAME=fa6f79b16832"],"namespaces":{"NEWIPC":true,"NEWNET":true,"NEWNS":true,"NEWPID":true,"NEWUTS":true},"capabilities":["CHOWN","DAC_OVERRIDE","FOWNER","MKNOD","NET_RAW","SETGID","SETUID","SETFCAP","SETPCAP","NET_BIND_SERVICE","SYS_CHROOT","KILL"],"networks":[{"type":"loopback","address":"127.0.0.1/0","gateway":"localhost","mtu":1500},{"type":"veth","context":{"bridge":"docker0","prefix":"veth"},"address":"172.17.0.3/16","gateway":"172.17.42.1","mtu":1500}],"cgroups":{"name":"fa6f79b1683273bb56257c57b7961f606b3f37be40ad14dc59d1ef394999a72d","parent":"docker","allowed_devices":[{"type":99,"major_number":-1,"minor_number":-1,"cgroup_permissions":"m"},{"type":98,"major_number":-1,"minor_number":-1,"cgroup_permissions":"m"},{"type":99,"path":"/dev/console","major_number":5,"minor_number":1,"cgroup_permissions":"rwm"},{"type":99,"path":"/dev/tty0","major_number":4,"cgroup_permissions":"rwm"},{"type":99,"path":"/dev/tty1","major_number":4,"minor_number":1,"cgroup_permissions":"rwm"},{"type":99,"major_number":136,"minor_number":-1,"cgroup_permissions":"rwm"},{"type":99,"major_number":5,"minor_number":2,"cgroup_permissions":"rwm"},{"type":99,"major_number":10,"minor_number":200,"cgroup_permissions":"rwm"},{"type":99,"path":"/dev/null","major_number":1,"minor_number":3,"cgroup_permissions":"rwm","file_mode":438},{"type":99,"path":"/dev/zero","major_number":1,"minor_number":5,"cgroup_permissions":"rwm","file_mode":438},{"type":99,"path":"/dev/full","major_number":1,"minor_number":7,"cgroup_permissions":"rwm","file_mode":438},{"type":99,"path":"/dev/tty","major_number":5,"cgroup_permissions":"rwm","file_mode":438},{"type":99,"path":"/dev/urandom","major_number":1,"minor_number":9,"cgroup_permissions":"rwm","file_mode":438},{"type":99,"path":"/dev/random","major_number":1,"minor_number":8,"cgroup_permissions":"rwm","file_mode":438}]},"context":{"apparmor_profile":"docker-default","mount_label":"","process_label":"","restrictions":"true"},"mounts":[{"type":"bind","source":"/var/lib/docker/init/dockerinit-1.0.0","destination":"/.dockerinit","private":true},{"type":"bind","source":"/var/lib/docker/containers/fa6f79b1683273bb56257c57b7961f606b3f37be40ad14dc59d1ef394999a72d/resolv.conf","destination":"/etc/resolv.conf","private":true},{"type":"bind","source":"/var/lib/docker/containers/fa6f79b1683273bb56257c57b7961f606b3f37be40ad14dc59d1ef394999a72d/hostname","destination":"/etc/hostname","private":true},{"type":"bind","source":"/var/lib/docker/containers/fa6f79b1683273bb56257c57b7961f606b3f37be40ad14dc59d1ef394999a72d/hosts","destination":"/etc/hosts","private":true},{"type":"bind","source":"/var/lib/docker/vfs/dir/4fb6c07d6ba799535bd0878c0706dcad2a47278e6ee703df37a9b829b5c7a9b2","destination":"/var/lib/redis","writable":true}],"device_nodes":[{"type":99,"path":"/dev/fuse","major_number":10,"minor_number":229,"cgroup_permissions":"rwm"},{"type":99,"path":"/dev/null","major_number":1,"minor_number":3,"cgroup_permissions":"rwm","file_mode":438},{"type":99,"path":"/dev/zero","major_number":1,"minor_number":5,"cgroup_permissions":"rwm","file_mode":438},{"type":99,"path":"/dev/full","major_number":1,"minor_number":7,"cgroup_permissions":"rwm","file_mode":438},{"type":99,"path":"/dev/tty","major_number":5,"cgroup_permissions":"rwm","file_mode":438},{"type":99,"path":"/dev/urandom","major_number":1,"minor_number":9,"cgroup_permissions":"rwm","file_mode":438},{"type":99,"path":"/dev/random","major_number":1,"minor_number":8,"cgroup_permissions":"rwm","file_mode":438}]} bash
root     26841 26660  0 10:57 pts/3    00:00:00 grep --color=auto nsinit
root@ubuntu:/#
root@ubuntu:/# pstree -p 26825
nsinit(26825)───bash(26830)
~~~

从上面的信息可以看出nsinit exec命令实际上是执行了nsinit nsenter命令。容器中的bash进程是nsinit进程的子进程（容器中的进程id为20，主机上的进程id为26830）。

退出bash进程后，nsinit进程随之退出。

#### 3.3 nsinit init

待补充

#### 3.4 nsinit stats

nsinit stats命令用于查看容器的状态信息，包括cpu、memory、blokio、freezer等内容。

~~~
root@ubuntu:/var/lib/docker/execdriver/native/fa6f79b1683273bb56257c57b7961f606b3f37be40ad14dc59d1ef394999a72d# /home/paas/libcontainer/nsinit stats
Stats:
{
	"cpu_stats": {
		"cpu_usage": {
			"percpu_usage": [
				484256561
			],
			"usage_in_kernelmode": 140000000,
			"usage_in_usermode": 160000000
		},
		"throlling_data": {}
	},
	"memory_stats": {
		"usage": 12230656,
		"max_usage": 12238848,
		"stats": {
			"active_anon": 6590464,
			"active_file": 0,
			"cache": 5652480,
			"hierarchical_memory_limit": 18446744073709551615,
			"hierarchical_memsw_limit": 18446744073709551615,
			"inactive_anon": 4096,
			"inactive_file": 5636096,
			"mapped_file": 0,
			"pgfault": 2081,
			"pgmajfault": 49,
			"pgpgin": 1795,
			"pgpgout": 342,
			"rss": 6578176,
			"rss_huge": 6291456,
			"swap": 0,
			"total_active_anon": 6590464,
			"total_active_file": 0,
			"total_cache": 5652480,
			"total_inactive_anon": 4096,
			"total_inactive_file": 5636096,
			"total_mapped_file": 0,
			"total_pgfault": 2081,
			"total_pgmajfault": 49,
			"total_pgpgin": 1795,
			"total_pgpgout": 342,
			"total_rss": 6578176,
			"total_rss_huge": 6291456,
			"total_swap": 0,
			"total_unevictable": 0,
			"total_writeback": 0,
			"unevictable": 0,
			"writeback": 0
		},
		"failcnt": 0
	},
	"blkio_stats": {},
	"freezer_stats": {
		"parent_state": "0",
		"self_state": "0"
	}
}
~~~

#### 3.5 nsinit spec

nsinit spec命令显示容器的详细信息。

~~~
root@ubuntu:/var/lib/docker/execdriver/native/fa6f79b1683273bb56257c57b7961f606b3f37be40ad14dc59d1ef394999a72d# /home/paas/libcontainer/nsinit spec
Spec:
{
	"hostname": "fa6f79b16832",
	"environment": [
		"HOME=/",
		"PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
		"HOSTNAME=fa6f79b16832"
	],
	"namespaces": {
		"NEWIPC": true,
		"NEWNET": true,
		"NEWNS": true,
		"NEWPID": true,
		"NEWUTS": true
	},
	"capabilities": [
		"CHOWN",
		"DAC_OVERRIDE",
		"FOWNER",
		"MKNOD",
		"NET_RAW",
		"SETGID",
		"SETUID",
		"SETFCAP",
		"SETPCAP",
		"NET_BIND_SERVICE",
		"SYS_CHROOT",
		"KILL"
	],
	"networks": [
		{
			"type": "loopback",
			"address": "127.0.0.1/0",
			"gateway": "localhost",
			"mtu": 1500
		},
		{
			"type": "veth",
			"context": {
				"bridge": "docker0",
				"prefix": "veth"
			},
			"address": "172.17.0.3/16",
			"gateway": "172.17.42.1",
			"mtu": 1500
		}
	],
	"cgroups": {
		"name": "fa6f79b1683273bb56257c57b7961f606b3f37be40ad14dc59d1ef394999a72d",
		"parent": "docker",
		"allowed_devices": [
			{
				"type": 99,
				"major_number": -1,
				"minor_number": -1,
				"cgroup_permissions": "m"
			},
			{
				"type": 98,
				"major_number": -1,
				"minor_number": -1,
				"cgroup_permissions": "m"
			},
			{
				"type": 99,
				"path": "/dev/console",
				"major_number": 5,
				"minor_number": 1,
				"cgroup_permissions": "rwm"
			},
			{
				"type": 99,
				"path": "/dev/tty0",
				"major_number": 4,
				"cgroup_permissions": "rwm"
			},
			{
				"type": 99,
				"path": "/dev/tty1",
				"major_number": 4,
				"minor_number": 1,
				"cgroup_permissions": "rwm"
			},
			{
				"type": 99,
				"major_number": 136,
				"minor_number": -1,
				"cgroup_permissions": "rwm"
			},
			{
				"type": 99,
				"major_number": 5,
				"minor_number": 2,
				"cgroup_permissions": "rwm"
			},
			{
				"type": 99,
				"major_number": 10,
				"minor_number": 200,
				"cgroup_permissions": "rwm"
			},
			{
				"type": 99,
				"path": "/dev/null",
				"major_number": 1,
				"minor_number": 3,
				"cgroup_permissions": "rwm",
				"file_mode": 438
			},
			{
				"type": 99,
				"path": "/dev/zero",
				"major_number": 1,
				"minor_number": 5,
				"cgroup_permissions": "rwm",
				"file_mode": 438
			},
			{
				"type": 99,
				"path": "/dev/full",
				"major_number": 1,
				"minor_number": 7,
				"cgroup_permissions": "rwm",
				"file_mode": 438
			},
			{
				"type": 99,
				"path": "/dev/tty",
				"major_number": 5,
				"cgroup_permissions": "rwm",
				"file_mode": 438
			},
			{
				"type": 99,
				"path": "/dev/urandom",
				"major_number": 1,
				"minor_number": 9,
				"cgroup_permissions": "rwm",
				"file_mode": 438
			},
			{
				"type": 99,
				"path": "/dev/random",
				"major_number": 1,
				"minor_number": 8,
				"cgroup_permissions": "rwm",
				"file_mode": 438
			}
		]
	},
	"context": {
		"apparmor_profile": "docker-default",
		"mount_label": "",
		"process_label": "",
		"restrictions": "true"
	},
	"mounts": [
		{
			"type": "bind",
			"source": "/var/lib/docker/init/dockerinit-1.0.0",
			"destination": "/.dockerinit",
			"private": true
		},
		{
			"type": "bind",
			"source": "/var/lib/docker/containers/fa6f79b1683273bb56257c57b7961f606b3f37be40ad14dc59d1ef394999a72d/resolv.conf",
			"destination": "/etc/resolv.conf",
			"private": true
		},
		{
			"type": "bind",
			"source": "/var/lib/docker/containers/fa6f79b1683273bb56257c57b7961f606b3f37be40ad14dc59d1ef394999a72d/hostname",
			"destination": "/etc/hostname",
			"private": true
		},
		{
			"type": "bind",
			"source": "/var/lib/docker/containers/fa6f79b1683273bb56257c57b7961f606b3f37be40ad14dc59d1ef394999a72d/hosts",
			"destination": "/etc/hosts",
			"private": true
		},
		{
			"type": "bind",
			"source": "/var/lib/docker/vfs/dir/4fb6c07d6ba799535bd0878c0706dcad2a47278e6ee703df37a9b829b5c7a9b2",
			"destination": "/var/lib/redis",
			"writable": true
		}
	],
	"device_nodes": [
		{
			"type": 99,
			"path": "/dev/fuse",
			"major_number": 10,
			"minor_number": 229,
			"cgroup_permissions": "rwm"
		},
		{
			"type": 99,
			"path": "/dev/null",
			"major_number": 1,
			"minor_number": 3,
			"cgroup_permissions": "rwm",
			"file_mode": 438
		},
		{
			"type": 99,
			"path": "/dev/zero",
			"major_number": 1,
			"minor_number": 5,
			"cgroup_permissions": "rwm",
			"file_mode": 438
		},
		{
			"type": 99,
			"path": "/dev/full",
			"major_number": 1,
			"minor_number": 7,
			"cgroup_permissions": "rwm",
			"file_mode": 438
		},
		{
			"type": 99,
			"path": "/dev/tty",
			"major_number": 5,
			"cgroup_permissions": "rwm",
			"file_mode": 438
		},
		{
			"type": 99,
			"path": "/dev/urandom",
			"major_number": 1,
			"minor_number": 9,
			"cgroup_permissions": "rwm",
			"file_mode": 438
		},
		{
			"type": 99,
			"path": "/dev/random",
			"major_number": 1,
			"minor_number": 8,
			"cgroup_permissions": "rwm",
			"file_mode": 438
		}
	]
}
~~~

#### 3.6 nsinit nsenter

在3.2中已经看到了nsinit nsenter的用法，不再重复介绍。

### 4 libcontainer

libcontainer由多个包构成，对于使用libcontainer的应用来说，最重要的包是namespaces，最重要的方法是namespaces.Init和namespace.Exec。

#### 4.1 namespaces.Init

dokcer中有两个可执行程序会调用namespaces.Init，一个是dockerinit-x.y.z（其中x.y.z为docker的版本号，位于/var/lib/docker/init/目录，此程序即每个容器中的.dockerinit程序），一个就是nsinit（在执行nsinit init命令时调用namespaces.Init）。

namespaces.Init的作用是对容器进行初始化操作。每个容器启动后，其中执行的第一个进程就是容器中的.dockerinit进程。在使用native方式启动容器时，.dockerinit调用namespaces.Init初始化容器。

namespaces.Init主要执行了以下工作：

1. 加载容器环境变量
2. setsid，即新建一个linux会话。
3. 设置网络
4. 设置路由
5. 初始化容器标签
6. 初始化mount命名空间
7. 设置apparmor
8. 设置命名空间
9. 使用exec运行应用程序

#### 4.2 namespaces.Exec

namespaces.Exec主要执行了以下工作：

1. 创建子进程（此时子进程处于阻塞状态）
2. 为子进程设置cgroup
3. 为子进程设置网络
4. 运行子进程
5. 等待子进程结束

#### 4.3 namespaces.Exec与namespaces.Init的关系

docker服务器使用libcontainer管理容器时，当用户通过docker客户端发出docker run命令运行容器时，处理过程是这样的：docker服务器做了已下操作：

1. docker服务器创建容器
2. docker服务器调用namespaces.Exec函数来运行.dockerinit进程
3. .dockerinit进程中调用namespaces.Init函数来初始化容器并运行用户指定的命令

即namespaces.Exec是docker服务器调用的函数，namespaces.Init是.dockerinit进程调用的函数。


[Docker之execdriver]: http://lsword.github.io/2014/06/04.html
[Docker之graphdriver]: http://lsword.github.io/2014/06/03.html
[Docker之网络模式]: http://lsword.github.io/2014/06/06.html