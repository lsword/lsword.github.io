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

在容器中运行一个命令。
	
### 4 libcontainer

未完待续...

[Docker之execdriver]: http://lsword.github.io/2014/06/04.html
[Docker之graphdriver]: http://lsword.github.io/2014/06/03.html
[Docker之网络模式]: http://lsword.github.io/2014/06/06.html