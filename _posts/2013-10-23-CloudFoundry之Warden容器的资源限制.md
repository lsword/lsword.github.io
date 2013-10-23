---
layout: post
category: 云计算
tags: CloudFoundry 容器 虚拟化
description: warden容器的主要目的是实现进程的资源隔离和限制，本文主要介绍warden容器中相关部分的实现。
---

### 术语
  1. warden主机：运行warden的主机，其上可以运行多个warden容器。
  2. warden容器：运行在warden主机上的虚拟主机，CloudFoundry的每个应用实例就运行在一个warden容器中。
  3. 应用实例：运行在warden容器中的一个应用。

### 内存

warden使用cgroup实现对进程内存的资源隔离和限制。

在warden/warden/lib/warden/container/features/mem_limit.rb中的do_limit_memory函数中实现。

dea可以通过接口控制warden设置对容器内存占用的限制。

do_limit_memory主要做以下两部分工作：

	启动oom进程，用于处理oom通知。
	向容器对应的cgroup目录中的memory.limit_in_bytes、memory.memsw.limit_in_bytes文件中写入内存上限值。

注意，在容器刚刚创建后是没有对于容器内存的限制的。可以通过手工执行warden命令接入warden服务器并使用create指令来创建一个容器，可以发现没有与其对应的oom进程。

用户通过cf stats appname命令查看进程运行信息时，warden通过读取memory.stat获取进程组的内存使用情况并返回。(在warden/warden/lib/warden/container/features/cgroup.rb中实现。)

#### cgroup中内存控制相关文件

	-rw-r--r-- cgroup.clone_children
	--w--w--w- cgroup.event_control
	-rw-r--r-- cgroup.procs
		用于设置和显示分组中的进程ID
	-rw-r--r-- memory.failcnt
		用于设置和显示分组内存占用达到限制值的次数
	--w------- memory.force_empty
		用于设置强制清空分组已用内存
	-rw-r--r-- memory.limit_in_bytes
		用于设置和显示分组的内存使用限量
	-rw-r--r-- memory.max_usage_in_bytes
		用于设置和显示分组的最大内存使用限量
	-rw-r--r-- memory.memsw.failcnt
		用于设置和显示分组内存+交换区占用达到限制值的次数
	-rw-r--r-- memory.memsw.limit_in_bytes
		用于设置和显示分组内存+交换区使用限量
	-rw-r--r-- memory.memsw.max_usage_in_bytes
		用于设置和显示分组内存+交换区使用限量
	-r--r--r-- memory.memsw.usage_in_bytes
		用于显示分组当前内存使用量+交换区使用量
	-rw-r--r-- memory.move_charge_at_immigrate
	-r--r--r-- memory.numa_stat
		用于显示分组当前numa(非统一内存访问)状态
	-rw-r--r-- memory.oom_control
		用于内存溢出通知和其他控制
		oom_kill_disable 如果为1，则禁用oom_killer
		under_oom 如果为1，则the memory cgroup is under OOM, tasks may be stopped
	-rw-r--r-- memory.soft_limit_in_bytes
		用于设置和显示分组内存+交换区使用软限制
	-r--r--r-- memory.stat
		用于显示内存使用统计数据
	-rw-r--r-- memory.swappiness
		用于设置和显示针对分组的swappiness
	-r--r--r-- memory.usage_in_bytes
		用于显示分组当前内存使用量
	-rw-r--r-- memory.use_hierarchy
		用于设置和显示层次结构数
	-rw-r--r-- notify_on_release
		默认为0。如果此值设置为1，则在分组中的最后一个任务和最后一个子分组被移除时，内核会调用cgroup根目录中release_agent文件中指定的命令。
	-rw-r--r-- tasks
		用于设置和显示分组中的进程、线程ID
	
#### cgroup中内存控制相关文件实例分析

在cf上启动一个ruby sinatra应用，设置占用128M内存。

	wshd(10074)───bash(10250)───startup(10254)───ruby(10258)─┬─startup(10259)───tee(10263)
															 ├─startup(10260)───tee(10262)
															 ├─{ruby}(10269)
															 └─{ruby}(10640)

	可以看出共有8个进程和2个ruby线程。

通过cf命令查看应用的内存占用量

	Getting stats for sinatra... OK

	instance   cpu    memory          disk
	#0         0.0%   17.7M of 128M   128.0M of 1G										
进入到应用相应的cgroup空间查看上述文件内容。

* cgroup.procs

	10074
	10250
	10254
	10258
	10259
	10260
	10262
	10263

	此处包含这个应用所有的进程号。
	
* memory.failcnt

	0

* memory.limit_in_bytes

	134217728 （128M）
	
	此值为应用内存使用上限。

* memory.max_usage_in_bytes

	102965248 （98.19M）

* memory.memsw.failcnt

	0

* memory.memsw.limit_in_bytes

	134217728 （128M）
	
	同memory.limit_in_bytes

* memory.memsw.max_usage_in_bytes

	102965248 （98.19M）
	
	同memory.max_usage_in_bytes

* memory.memsw.usage_in_bytes
	
	36577280 （34.88M）
	
* memory.move_charge_at_immigrate

	0

* memory.numa_stat

	total=7452 N0=7452
	
	file=2912 N0=2912
	
	anon=4540 N0=4540
	
	unevictable=0 N0=0
	
* memory.oom_control

	oom_kill_disable 0
	
	under_oom 0

* memory.soft_limit_in_bytes

	9223372036854775807

* memory.stat

	cache 11927552	（页面缓存量）
	
	rss 18604032	（物理内存使用量，与用cf命令查出应用内存占用量是一致的）
	
	mapped_file 2297856		(指向进程空间的文件映射所使用的内存量)
	
	swap 6045696	(交换区使用量)
	
	pgpgin 56126	(页面换入次数)
	
	pgpgout 48672	(页面换出次数)
	
	pgfault 37372	(二级页面错误数)
	
	pgmajfault 1330		(一级页面错误数)
	
	inactive_anon 9461760	(LRU列表中无效的匿名页面数字节数)
	
	active_anon 9142272		(LRU列表中有效的匿名页面数字节数)
	
	inactive_file 5505024	(LRU列表中无效的文件字节缓存数)
	
	active_file 6422528		(LRU列表中有效的文件字节缓存数)
	
	unevictable 0	(不能用mlock等回收的内存量)
	
	hierarchical_memory_limit 134217728	(上层分组对内存的限制)
	
	hierarchical_memsw_limit 134217728		(上层分组对内存+交换区的限制)
	
	total_cache 11927552	（本分组所有页面缓存量）
	
	total_rss 18604032	(本分组所有物理内存使用量)
	
	total_mapped_file 2297856	(本分组所有指向进程空间的文件映射所使用的内存量)
	
	total_swap 6045696		(本分组所有交换区使用量)
	
	total_pgpgin 56126		(本分组所有页面换入次数)
	
	total_pgpgout 48672		(本分组所有页面换出次数)
	
	total_pgfault 37372		(本分组所有二级页面错误数)
	
	total_pgmajfault 1330	(本分组所有一级页面错误数)
	
	total_inactive_anon 9461760		(本分组所有LRU列表中无效的匿名页面数字节数)
	
	total_active_anon 9142272	(本分组所有LRU列表中有效的匿名页面数字节数)
	
	total_inactive_file 5505024		(本分组所有LRU列表中无效的文件字节缓存数)
	
	total_active_file 6422528	(本分组所有LRU列表中有效的文件字节缓存数)
	
	total_unevictable 0		(本分组所有不能用mlock等回收的内存量)


* memory.swappiness

	60
	
	同vm.swappiness。	
	不是等所有的物理内存都消耗完毕之后，才去使用swap的空间，什么时候使用是由swappiness参数值控制。
	swappiness=0的时候表示最大限度使用物理内存，然后才是swap空间，
	swappiness＝100的时候表示积极的使用swap分区，并且把内存上的数据及时的搬运到swap空间里面。
	
* memory.usage_in_bytes

	30523392 （29.11M）

	此值为应用已使用内存。
	
* memory.use_hierarchy

	0
	
* notify_on_release

	0
	
* tasks

	10074
	10250
	10254
	10258
	10259
	10260
	10262
	10263
	10269
	10640

	此处包含这个应用所有的进程号和线程号。
	
#### oom处理

memory.oom_control中的oom_kill_disable值为0，表示分组中进程内存使用超出限制时，会通过oom killer机制杀掉相应的进程。

warden容器配套的oom进程会监听容器中进程被oom killer机制杀掉的事件，通知warden服务器进程是由于内存占用过多导致。

### CPU

warden使用cgroup实现对进程cpu的资源隔离和限制。

warden在这方面都使用默认值，不向外提供控制接口。

#### cgroup中CPU控制相关文件

	-rw-r--r-- cgroup.clone_children
	
	--w--w--w- cgroup.event_control
	
	-rw-r--r-- cgroup.procs
		用于设置和显示分组中的进程ID
		
	-rw-r--r-- cpu.cfs_period_us
		cfs(Completely Fair Scheduler 完全公平调度)调度方法的运行周期长度(微秒)。
		参考[sched-bwc]
		
	-rw-r--r-- cpu.cfs_quota_us
		cfs(Completely Fair Scheduler 完全公平调度)调度方法的一个周期内的可用运行时间(微秒)。
		-1表示没有任何cfs带宽限制。
		参考[sched-bwc]
		
	-rw-r--r-- cpu.rt_period_us
		rt(Real-Time group scheduling 实时组调度)调度方法的运行周期长度(微秒)。
		参考[sched-rt-group]
		
	-rw-r--r-- cpu.rt_runtime_us
		rt(Real-Time group scheduling 实时组调度)调度方法中为此进程组保留的cpu运行时间(微秒)。
		参考[sched-rt-group]
	
	-rw-r--r-- cpu.shares
		cpu时间占用比例。
	
	-r--r--r-- cpu.stat
		cpu状态。
	
	-rw-r--r-- notify_on_release
		默认为0。如果此值设置为1，则在分组中的最后一个任务和最后一个子分组被移除时，内核会调用cgroup根目录中release_agent文件中指定的命令。
		
	-rw-r--r-- tasks
		用于设置和显示分组中的进程、线程ID
		
#### cgroup中CPU控制相关文件实例分析

在cf上启动一个ruby sinatra应用，设置占用128M内存。

	wshd(10074)───bash(10250)───startup(10254)───ruby(10258)─┬─startup(10259)───tee(10263)
															 ├─startup(10260)───tee(10262)
															 ├─{ruby}(10269)
															 └─{ruby}(10640)

	可以看出共有8个进程和2个ruby线程。

通过cf命令查看应用的cpu占用量

	Getting stats for sinatra... OK

	instance   cpu    memory          disk
	#0         0.0%   17.7M of 128M   128.0M of 1G										
进入到应用相应的cgroup空间查看上述文件内容。

* cgroup.procs

	10074
	10250
	10254
	10258
	10259
	10260
	10262
	10263

	此处包含这个应用所有的进程号。
	
* cpu.cfs_period_us

	100000

* cpu.cfs_quota_us

	-1

* cpu.rt_period_us

	1000000

* cpu.rt_runtime_us

	0

* cpu.shares

	1024

* cpu.stat

	nr_periods 0
	
	nr_throttled 0
	
	throttled_time 0

* notify_on_release

	0
	
* tasks

	10074
	10250
	10254
	10258
	10259
	10260
	10262
	10263
	10269
	10640

	此处包含这个应用所有的进程号和线程号。
	
### CPUACCT

warden使用cgroup实现对进程cpuacct的资源隔离和限制。

warden在这方面都使用默认值，不向外提供控制接口。

用户通过cf stats appname命令查看进程运行信息时，warden通过读取cpuacct.usage和cpuacct.stat获取进程组的CPU占用情况并返回。(在warden/warden/lib/warden/container/features/cgroup.rb中实现。)

#### cgroup中CPUACCT控制相关文件

	-rw-r--r-- cgroup.clone_children
	
	--w--w--w- cgroup.event_control
	
	-rw-r--r-- cgroup.procs
		用于设置和显示分组中的进程ID
		
	-r--r--r-- cpuacct.stat
		用于查看分组中进程在用户模式和内核模式的消耗时间（单位：USER_HZ）。
		
	-rw-r--r-- cpuacct.usage
		用于查看分组中进程消耗的所有的cpu时间（单位：纳秒）
		
	-rw-r--r-- cpuacct.usage_percpu
		用于查看分组中进程消耗的所有cpu的时间（单位：纳秒）
		
	-rw-r--r-- notify_on_release
		默认为0。如果此值设置为1，则在分组中的最后一个任务和最后一个子分组被移除时，内核会调用cgroup根目录中release_agent文件中指定的命令。
		
	-rw-r--r-- tasks
		用于设置和显示分组中的进程、线程ID
		
#### cgroup中CPU控制相关文件实例分析

在cf上启动一个ruby sinatra应用，设置占用128M内存。

	wshd(10074)───bash(10250)───startup(10254)───ruby(10258)─┬─startup(10259)───tee(10263)
															 ├─startup(10260)───tee(10262)
															 ├─{ruby}(10269)
															 └─{ruby}(10640)

	可以看出共有8个进程和2个ruby线程。

通过cf命令查看应用的cpu占用量

	Getting stats for sinatra... OK

	instance   cpu    memory          disk
	#0         0.0%   17.7M of 128M   128.0M of 1G										
进入到应用相应的cgroup空间查看上述文件内容。

* cgroup.procs

	10074
	10250
	10254
	10258
	10259
	10260
	10262
	10263

	此处包含这个应用所有的进程号。
	
* cpuacct.stat

	user 1872
	
	system 512

* cpuacct.usage

	34450199646

* cpuacct.usage_percpu

	34450199646
	
	此应用实例只在一个cpu上运行，因此此值与cpuacct.usage相同。
	
* notify_on_release

	0
	
* tasks

	10074
	10250
	10254
	10258
	10259
	10260
	10262
	10263
	10269
	10640

	此处包含这个应用所有的进程号和线程号。
	
	
### 设备

warden使用cgroup实现对进程可访问设备的限制。

warden在创建容器脚本(warden/warden/root/linux/skeleton/setup.sh)中将容器目标文件系统中/dev目录下所有的设备文件清空，只创建几个部分字符设备用于应用访问。在cgroup中devices.list中的设置与这些字符设备对应.

#### cgroup中DEVICE控制相关文件

	-rw-r--r-- cgroup.clone_children
	
	--w--w--w- cgroup.event_control
	
	-rw-r--r-- cgroup.procs
		用于设置和显示分组中的进程ID
		
	--w------- devices.allow
		用于设置分组中进程可访问设备
	
	--w------- devices.deny
		用于设置分组中进程不可访问设备
	
	-r--r--r-- devices.list
		用于查看分组中进程可访问设备
	
	-rw-r--r-- notify_on_release
		默认为0。如果此值设置为1，则在分组中的最后一个任务和最后一个子分组被移除时，内核会调用cgroup根目录中release_agent文件中指定的命令。
		
	-rw-r--r-- tasks
		用于设置和显示分组中的进程、线程ID
	
#### cgroup中DEVICE控制相关文件实例分析

在cf上启动一个ruby sinatra应用，设置占用128M内存。

	wshd(10074)───bash(10250)───startup(10254)───ruby(10258)─┬─startup(10259)───tee(10263)
															 ├─startup(10260)───tee(10262)
															 ├─{ruby}(10269)
															 └─{ruby}(10640)

	可以看出共有8个进程和2个ruby线程。

进入到应用相应的cgroup空间查看上述文件内容。

* cgroup.procs

	10074
	10250
	10254
	10258
	10259
	10260
	10262
	10263

	此处包含这个应用所有的进程号。
	
* cpuacct.stat

	c 1:3 rw	(/dev/null)
		
	c 1:5 rw	(/dev/zero)
	
	c 1:8 rw	(/dev/random)
	
	c 1:9 rw	(/dev/urandom)
	
	c 5:0 rw	(/dev/tty)
	
	c 5:2 rw	(/dev/pts/ptms)
	
	c 136:* rw	(/dev/pts/0、/dev/pts/1、...)

	格式	：设备类型 主设备号:从设备号 读写权限
	
	可以通过wsh命令接入到warden容器内部，查看容器中/dev目录下的相应文件。
	
* notify_on_release

	0
	
* tasks

	10074
	10250
	10254
	10258
	10259
	10260
	10262
	10263
	10269
	10640

	此处包含这个应用所有的进程号和线程号。
	
### 磁盘

warden使用quota来实现对容器磁盘空间占用的限制。

在warden/warden/lib/warden/container/features/quota.rb中的do_limit_disk函数中实现。

dea可以通过接口控制warden设置对容器磁盘占用的限制。注意，在容器刚刚创建后是没有对于容器磁盘的限制的。

主要根据外部请求计算下面四个参数：

	limits[:block_soft]		磁盘块占用的软限制
	limits[:block_hard]		磁盘块占用的硬限制
	limits[:inode_soft]		inode占用的软限制
	limits[:inode_hard]		inode占用的硬限制
	
计算完成后，调用shell命令setquota来设置容器目录的磁盘空间限制。


### 网络

warden使用linux中的tc程序设置网络流量控制参数。

dea控制warden服务器创建容器运行客户应用时，不进行网络流量设置。只是在容器初始化时进行缺省设置。

配置示例：

	qdisc pfifo_fast 0: dev eth0 root refcnt 2 bands 3 priomap  1 2 2 2 1 2 0 0 1 1 1 1 1 1 1 1
	qdisc pfifo_fast 0: dev w-178l4d6v637-0 root refcnt 2 bands 3 priomap  1 2 2 2 1 2 0 0 1 1 1 1 1 1 1 1
	qdisc pfifo_fast 0: dev w-1797f48b5cp-0 root refcnt 2 bands 3 priomap  1 2 2 2 1 2 0 0 1 1 1 1 1 1 1 1
	
从上面可以看出，每个容器的流控参数设置相同。

warden对外提供了网络流量控制设置接口，在warden/warden/lib/warden/container/features/net.rb中的do_limit_bandwidth函数中实现。

此函数调用容器中的net_rate.sh脚本设置网络带宽：

	# clear rule if exist
	# delete root egress tc qdisc
	tc qdisc del dev ${network_host_iface} root 2> /dev/null || true

	# delete root ingress tc qdisc
	tc qdisc del dev ${network_host_iface} ingress 2> /dev/null || true

	# set inbound(outside -> eth0 -> w-<cid>-0 -> w-<cid>-1) rule with tc's tbf(token bucket filter) qdisc
	# rate is the bandwidth
	# burst is the burst size
	# latency is the maxium time the packet wait to enqueue while no token left
	tc qdisc add dev ${network_host_iface} root tbf rate ${RATE}bit burst ${BURST} latency 25ms

	# set outbound(w-<cid>-1 -> w-<cid>-0 -> eth0 -> outside)  rule
	tc qdisc add dev ${network_host_iface} ingress handle ffff:

	# use u32 filter with target(0.0.0.0) mask (0) to filter all the ingress packets
	tc filter add dev ${network_host_iface} parent ffff: protocol ip prio 1 u32 match ip src 0.0.0.0/0 police rate ${RATE}bit burst ${BURST} drop flowid :1


可以使用shell命令(tc -s -d qdisc show dev 网卡设备)来查看tc状态。
	

[Linux中的namespaces]: http://lsword.github.io/2013/09/20.html
[sched-bwc]: https://www.kernel.org/doc/Documentation/scheduler/sched-bwc.txt
[sched-rt-group]: https://www.kernel.org/doc/Documentation/scheduler/sched-rt-group.txt