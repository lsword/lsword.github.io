---
layout: post
category: 云计算 虚拟化
tags: docker
description: 本文介绍docker容器中实现容器运行管理的execdriver
---

### 1、背景

本文中涉及的源码基于docker 0.11.1版本系统。
本文中涉及的测试环境基于ubuntu server 14.04及redhat6.5。

docker对于容器运行的管理也是可扩展的，通过execdriver实现。

### 2、概述

execdriver源码位于docker/daemon目录下。

driver.go文件中定义了Driver的接口：

~~~
Run						运行容器
Kill					向容器发送信号
Name					获取驱动的名称和版本
Info					获取驱动信息
GetPidsForContainer		获取容器中的pid
Terminate				强制终止容器运行（kill -9）
~~~
在lxc和native中分别对这些函数进行了实现。

docker目前支持lxc和native两种容器运行管理方式。在redhat6.5环境下使用的运行管理驱动是lxc，在ubuntu14.04环境下使用的运行管理驱动是native。

### 3 lxc

lxc是在Linux环境下实现基于操作系统的轻量级虚拟化的工具软件，具体情况可参考其官方文档。下面主要介绍docker是如何使用lxc来实现容器运行管理驱动的。


#### 3.1 Run

使用lxc运行容器，主要需要进行以下工作：

~~~
准备rootfs
生成lxc配置文件
执行lxc-start命令启动容器
~~~

其中准备rootfs的工作是有graphdriver实现的，具体情况请参考[Docker之graphdriver]。

##### 3.1.1 lxc配置文件

lxc的配置文件保存在/var/lib/docker/containers/容器id/config.lxc中。

已下为一个简单docker容器的lxc配置文件

~~~
# network configuration
lxc.network.type = veth
lxc.network.link = docker0
lxc.network.name = eth0
lxc.network.mtu = 1500


# root filesystem

lxc.rootfs = /var/lib/docker/containers/5c750830982f852cae616ac7408823f716efcce02bf95bbbb926abb3788f30a6/root

# use a dedicated pts for the container (and limit the number of pseudo terminal
# available)
lxc.pts = 1024

# disable the main console
lxc.console = none

# no controlling tty at all
lxc.tty = 1


# no implicit access to devices
lxc.cgroup.devices.deny = a

# /dev/null and zero
lxc.cgroup.devices.allow = c 1:3 rwm
lxc.cgroup.devices.allow = c 1:5 rwm

# consoles
lxc.cgroup.devices.allow = c 5:1 rwm
lxc.cgroup.devices.allow = c 5:0 rwm
lxc.cgroup.devices.allow = c 4:0 rwm
lxc.cgroup.devices.allow = c 4:1 rwm

# /dev/urandom,/dev/random
lxc.cgroup.devices.allow = c 1:9 rwm
lxc.cgroup.devices.allow = c 1:8 rwm

# /dev/pts/ - pts namespaces are "coming soon"
lxc.cgroup.devices.allow = c 136:* rwm
lxc.cgroup.devices.allow = c 5:2 rwm

# tuntap
lxc.cgroup.devices.allow = c 10:200 rwm

# fuse
#lxc.cgroup.devices.allow = c 10:229 rwm

# rtc
#lxc.cgroup.devices.allow = c 254:0 rwm


# standard mount point
# Use mnt.putold as per https://bugs.launchpad.net/ubuntu/+source/lxc/+bug/986385
lxc.pivotdir = lxc_putold

# NOTICE: These mounts must be applied within the namespace

#  WARNING: procfs is a known attack vector and should probably be disabled
#           if your userspace allows it. eg. see http://blog.zx2c4.com/749
lxc.mount.entry = proc /var/lib/docker/containers/5c750830982f852cae616ac7408823f716efcce02bf95bbbb926abb3788f30a6/root/proc proc nosuid,nodev,noexec 0 0

# WARNING: sysfs is a known attack vector and should probably be disabled
# if your userspace allows it. eg. see http://bit.ly/T9CkqJ
lxc.mount.entry = sysfs /var/lib/docker/containers/5c750830982f852cae616ac7408823f716efcce02bf95bbbb926abb3788f30a6/root/sys sysfs nosuid,nodev,noexec 0 0


lxc.mount.entry = /dev/pts/2 /var/lib/docker/containers/5c750830982f852cae616ac7408823f716efcce02bf95bbbb926abb3788f30a6/root/dev/console none bind,rw 0 0


lxc.mount.entry = devpts /var/lib/docker/containers/5c750830982f852cae616ac7408823f716efcce02bf95bbbb926abb3788f30a6/root/dev/pts devpts newinstance,ptmxmode=0666,nosuid,noexec 0 0
lxc.mount.entry = shm /var/lib/docker/containers/5c750830982f852cae616ac7408823f716efcce02bf95bbbb926abb3788f30a6/root/dev/shm tmpfs size=65536k,nosuid,nodev,noexec 0 0
~~~

##### 3.1.2 lxc-start

以下为一个典型的容器启动命令。

lxc-start -n 5c750830982f852cae616ac7408823f716efcce02bf95bbbb926abb3788f30a6 -f /var/lib/docker/containers/5c750830982f852cae616ac7408823f716efcce02bf95bbbb926abb3788f30a6/config.lxc -- /.dockerinit -driver lxc -g 172.17.42.1 -i 172.17.0.2/16 -mtu 1500 -- bash

从上面的命令可以看出是使用lxc-start启动容器，在容器中运行/.dockerinit -driver lxc -g 172.17.42.1 -i 172.17.0.2/16 -mtu 1500 -- bash命令，由.dockerinit进程启动bash进程。

以下为启动一个容器后的docker进程树示例：

~~~

[root@localhost paas]# pstree -p 2772
docker(2772)─┬─lxc-start(3450)───bash(3455)
             ├─{docker}(2774)
             ├─{docker}(2775)
             ├─{docker}(2776)
             ├─{docker}(2777)
             ├─{docker}(2778)
             ├─{docker}(2780)
             ├─{docker}(2886)
             └─{docker}(2904)
~~~

从代码中可以看出，容器初始化工作由.dockerinit进程完成，初始化结束后，.dockerinit进程通过exec系统调用运行bash进程。

#### 3.2 Kill

执行以下命令

lxc-kill -n 容器id 信号

#### 3.3 Name

返回lxc驱动的名称和版本。

#### 3.4 Info

返回lxc驱动的信息。

#### 3.5 GetPidsForContainer

从cgroup/cpu/lxc/容器id/tasks文件中读取pid信息并返回。 

#### 3.6 Terminate

执行以下命令

lxc-kill -n 容器id 9

### 4 native

#### 4.1 Run

使用native方式运行容器，主要需要进行以下工作：

~~~
创建容器数据结构
准备native容器目录
准备native容器配置文件
启动容器
~~~

在这种方式下，使用docker自带的libcontainer库来管理容器，关于libcontainer的详细情况请参考[Docker之libcontainer]。

##### 4.1.1 创建容器数据结构

定义在docker/pkg/libcontainer/container.go文件中。

~~~
// Context is a generic key value pair that allows
// arbatrary data to be sent
type Context map[string]string

// Container defines configuration options for how a
// container is setup inside a directory and how a process should be executed
type Container struct {
	Hostname         string          `json:"hostname,omitempty"`          // hostname
	ReadonlyFs       bool            `json:"readonly_fs,omitempty"`       // set the containers rootfs as readonly
	NoPivotRoot      bool            `json:"no_pivot_root,omitempty"`     // this can be enabled if you are running in ramdisk
	User             string          `json:"user,omitempty"`              // user to execute the process as
	WorkingDir       string          `json:"working_dir,omitempty"`       // current working directory
	Env              []string        `json:"environment,omitempty"`       // environment to set
	Tty              bool            `json:"tty,omitempty"`               // setup a proper tty or not
	Namespaces       map[string]bool `json:"namespaces,omitempty"`        // namespaces to apply
	CapabilitiesMask map[string]bool `json:"capabilities_mask,omitempty"` // capabilities to drop
	Networks         []*Network      `json:"networks,omitempty"`          // nil for host's network stack
	Cgroups          *cgroups.Cgroup `json:"cgroups,omitempty"`           // cgroups
	Context          Context         `json:"context,omitempty"`           // generic context for specific options (apparmor, selinux)
	Mounts           Mounts          `json:"mounts,omitempty"`
}

// Network defines configuration for a container's networking stack
//
// The network configuration can be omited from a container causing the
// container to be setup with the host's networking stack
type Network struct {
	Type    string  `json:"type,omitempty"`    // type of networking to setup i.e. veth, macvlan, etc
	Context Context `json:"context,omitempty"` // generic context for type specific networking options
	Address string  `json:"address,omitempty"`
	Gateway string  `json:"gateway,omitempty"`
	Mtu     int     `json:"mtu,omitempty"`
}

~~~

##### 4.1.2 准备native容器目录
创建以下目录
/var/lib/docker/execdriver/native/容器id

##### 4.1.3 准备native容器配置文件
创建以下文件
/var/lib/docker/execdriver/native/容器id/container.json

示例

~~~
{
	"hostname":"236cd8e98911",
	"environment":[
		"HOME=/",
		"PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
		"HOSTNAME=236cd8e98911"
	],
	"namespaces":{
		"NEWIPC":true,
		"NEWNET":true,
		"NEWNS":true,
		"NEWPID":true,
		"NEWUTS":true
	},
	"capabilities_mask":{
		"AUDIT_CONTROL":false,
		"AUDIT_WRITE":false,
		"MAC_ADMIN":false,
		"MAC_OVERRIDE":false,
		"MKNOD":true,
		"NET_ADMIN":false,
		"SETPCAP":false,
		"SYSLOG":false,
		"SYS_ADMIN":false,
		"SYS_MODULE":false,
		"SYS_NICE":false,
		"SYS_PACCT":false,
		"SYS_RAWIO":false,
		"SYS_RESOURCE":false,
		"SYS_TIME":false,
		"SYS_TTY_CONFIG":false
	},
	"networks":[
		{
			"type":"loopback",
			"address":"127.0.0.1/0",
			"gateway":"localhost",
			"mtu":1500
		},
		{
			"type":"veth",
			"context":{"bridge":"docker0","prefix":"veth"},
			"address":"172.17.0.2/16",
			"gateway":"172.17.42.1",
			"mtu":1500}
	],
	"cgroups":{
		"name":"236cd8e989112ba83f935d64aa84da6fcc5a209b6dcdfd30f4937d93cf293b40",
		"parent":"docker"
	},
	"context":{
		"apparmor_profile":"docker-default",
		"mount_label":"",
		"process_label":"",
		"restrictions":"true"},
	"mounts":[
		{"type":"devtmpfs"},
		{"type":"bind","source":"/var/lib/docker/init/dockerinit-0.11.1","destination":"/.dockerinit","private":true},
		{"type":"bind","source":"/var/lib/docker/containers/236cd8e989112ba83f935d64aa84da6fcc5a209b6dcdfd30f4937d93cf293b40/resolv.conf","destination":"/etc/resolv.conf","private":true},
		{"type":"bind","source":"/var/lib/docker/containers/236cd8e989112ba83f935d64aa84da6fcc5a209b6dcdfd30f4937d93cf293b40/hostname","destination":"/etc/hostname","private":true},
		{"type":"bind","source":"/var/lib/docker/containers/236cd8e989112ba83f935d64aa84da6fcc5a209b6dcdfd30f4937d93cf293b40/hosts","destination":"/etc/hosts","private":true},
		{"type":"bind","source":"/var/lib/docker/vfs/dir/d5e248d4c4a50e8b650fc1d38e278dbdc8c0f66939ee85fb72382743734c46d9","destination":"/var/lib/redis","writable":true}
	]
}
~~~

这个文件的内容是根据第一步创建的容器数据结构生成的。

##### 4.1.4 启动容器

调用libcontainer库中的namespaces.Exec函数，在容器中启动.dockerinit进程，由.dockerinit进程完成初始化操作，并启动容器中的应用程序。

#### 4.2 Kill

执行kill系统调用，向容器进行发送信号。

#### 4.3 Name

返回native驱动的名称和版本。

#### 4.4 Info

返回native驱动的信息。

#### 4.5 GetPidsForContainer

从/sys/fs/cgroup/devices/docker/容器id/tasks文件中读取pid信息并返回。 

#### 4.6 Terminate

执行kill系统调用，向容器进行发送信号9。


[Docker之libcontainer]: http://lsword.github.io/2014/06/16.html
[Docker之graphdriver]: http://lsword.github.io/2014/06/03.html
[thinprovisioned_volumes]: https://access.redhat.com/site/documentation/en-US/Red_Hat_Enterprise_Linux/6/html/Logical_Volume_Manager_Administration/thinprovisioned_volumes.html
[device-mapper]: http://device-mapper.com/
[thin-provisioning.txt]:https://github.com/torvalds/linux/blob/master/Documentation/device-mapper/thin-provisioning.txt