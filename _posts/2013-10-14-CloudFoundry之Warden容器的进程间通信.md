---
layout: post
category: 云计算
tags: CloudFoundry 容器 虚拟化
description: 一个warden容器会涉及到一系列的进程，本文对warden容器中的进程结构和进程间通信进行介绍。
---

### 术语
  1. warden主机：运行warden的主机，其上可以运行多个warden容器。
  2. warden容器：运行在warden主机上的虚拟主机，CloudFoundry的每个应用实例就运行在一个warden容器中。
  3. 应用实例：运行在warden容器中的一个应用。

### warden容器的进程结构

弄清warden进程间通信前，需要先了解warden容器的进程结构。虽然warden可以独立运行，但是目前要真正发挥作用，还是需要放到cloudfoundry这个大环境中，所以下面以一个实际运行cloudfoundry环境中的warden容器来进行介绍。

启动cloudfoundry后，通过ps命令找到warden服务器进程，再通过pstree命令查看其进程情况，结果如下：

	paas@ubuntu:~/paas-release.git/bin$ ps -ef | grep warden
	root     13828     1  0 09:06 pts/0    00:00:00 sudo /home/paas/build_dir/target/ruby/bin/ruby /home/paas/build_dir/target/ruby/bin/bundle exec /home/paas/build_dir/target/ruby/bin/ruby /home/paas/build_dir/target/ruby/bin/rake --trace warden:start[/home/paas/vcap/sys/config/warden.yml]
	root     13832 13828  0 09:06 pts/0    00:00:02 /home/paas/build_dir/target/ruby/bin/ruby /home/paas/build_dir/target/ruby/bin/rake --trace warden:start[/home/paas/vcap/sys/config/warden.yml]
	root     14306 13832  0 09:08 pts/0    00:00:00 /home/paas/build_dir/target/warden_2fd4fcd33.git/warden/src/oom/oom /tmp/warden/cgroup/memory/instance-178l4d6v636
	root     14909 13832  0 09:14 pts/0    00:00:00 /bin/bash /home/paas/warden/container/178l4d6v636/stop.sh
	paas     14965 13616  0 09:14 pts/0    00:00:00 grep --color=auto warden
	paas@ubuntu:~/paas-release.git/bin$
	paas@ubuntu:~/paas-release.git/bin$ pstree -p 13832
	ruby(13832)─┬─{ruby}(14003)
	            ├─{ruby}(14319)
	            ├─{ruby}(14320)
	            ├─{ruby}(14321)
	            ├─{ruby}(14322)
	            ├─{ruby}(14323)
	            ├─{ruby}(14324)
	            ├─{ruby}(14325)
	            ├─{ruby}(14326)
	            ├─{ruby}(14327)
	            ├─{ruby}(14328)
	            ├─{ruby}(14329)
	            ├─{ruby}(14330)
	            ├─{ruby}(14331)
	            ├─{ruby}(14332)
	            ├─{ruby}(14333)
	            ├─{ruby}(14334)
	            ├─{ruby}(14335)
	            ├─{ruby}(14336)
	            ├─{ruby}(14337)
	            └─{ruby}(14338)
        	    
此时只看到了一个ruby进程和其下的一系列线程，此时没有应用运行。

通过cf命令启动一个测试应用后，通过pstree命令查看其进程情况，结果如下：


	paas@ubuntu:~/paas-release.git/bin$ cf start sinatra
	Starting sinatra... OK
	Checking sinatra...
	  0/1 instances: 1 starting
	  0/1 instances: 1 starting
	  0/1 instances: 1 starting
	  0/1 instances: 1 starting
	  0/1 instances: 1 starting
	  0/1 instances: 1 starting
	  0/1 instances: 1 starting
	  0/1 instances: 1 starting
	  1/1 instances: 1 running
	OK
	paas@ubuntu:~/paas-release.git/bin$ pstree -p 13832
	ruby(13832)─┬─iomux-link(15289)
	            ├─iomux-spawn(15282)─┬─wsh(15283)
	            │                    ├─{iomux-spawn}(15284)
	            │                    ├─{iomux-spawn}(15285)
	            │                    ├─{iomux-spawn}(15286)
	            │                    ├─{iomux-spawn}(15287)
	            │                    └─{iomux-spawn}(15288)
	            ├─oom(15194)
	            ├─{ruby}(14003)
	            ├─{ruby}(14319)
	            ├─{ruby}(14320)
	            ├─{ruby}(14321)
	            ├─{ruby}(14322)
	            ├─{ruby}(14323)
	            ├─{ruby}(14324)
	            ├─{ruby}(14325)
	            ├─{ruby}(14326)
	            ├─{ruby}(14327)
	            ├─{ruby}(14328)
	            ├─{ruby}(14329)
	            ├─{ruby}(14330)
	            ├─{ruby}(14331)
	            ├─{ruby}(14332)
	            ├─{ruby}(14333)
	            ├─{ruby}(14334)
	            ├─{ruby}(14335)
	            ├─{ruby}(14336)
	            ├─{ruby}(14337)
	            └─{ruby}(14338)

可以看出warden服务进程启动了三个子进程iomux-link、iomux-spawn、oom。其中iomux-spawn进程又有一个子进程wsh和5个线程。

此时已经创建了一个warden容器，通过ps和pstree命令查看相应的wshd进程，结果如下：

	paas@ubuntu:~/paas-release.git/bin$ ps -ef | grep wshd
	root     15141     1  0 09:18 ?        00:00:00 wshd: 178l4d6v637
	root     15282 13832  0 09:19 ?        00:00:00 /home/paas/warden/container/178l4d6v637/bin/iomux-spawn /home/paas/warden/container/178l4d6v637/jobs/6 /home/paas/warden/container/178l4d6v637/bin/wsh --socket /home/paas/warden/container/178l4d6v637/run/wshd.sock --user vcap /bin/bash
	root     15283 15282  0 09:19 ?        00:00:00 /home/paas/warden/container/178l4d6v637/bin/wsh --socket /home/paas/warden/container/178l4d6v637/run/wshd.sock --user vcap /bin/bash
	paas     15996 13616  0 09:27 pts/0    00:00:00 grep --color=auto wshd
	paas@ubuntu:~/paas-release.git/bin$ pstree -p 15141
	wshd(15141)───bash(15292)───startup(15304)───ruby(15306)─┬─startup(15307)───tee(15310)
	                                                         ├─startup(15308)───tee(15309)
	                                                         └─{ruby}(15312)

可以看到，已经存在了一个wshd进程，其下有一系列子进程。其中bash是响应wsh的请求创建的shell进程，startup是应用的启动脚本程序（dea在进行应用打包时，为每个应用都生成一个startup脚本用来启动应用），ruby(15306)是应用进程。

此时通过cf命令修改应用实例个数为2，再次查看warden服务进程的情况，结果如下：

	paas@ubuntu:~/paas-release.git/bin$ cf scale sinatra
	Instances> 2

	1: 128M
	2: 256M
	3: 512M
	4: 1G
	Memory Limit> 128M

	Scaling sinatra... OK

	paas@ubuntu:~/paas-release.git$ pstree -p 13832
	ruby(13832)─┬─iomux-link(15289)
	            ├─iomux-link(20047)
	            ├─iomux-spawn(15282)─┬─wsh(15283)
	            │                    ├─{iomux-spawn}(15284)
	            │                    ├─{iomux-spawn}(15285)
	            │                    ├─{iomux-spawn}(15286)
	            │                    ├─{iomux-spawn}(15287)
	            │                    └─{iomux-spawn}(15288)
	            ├─iomux-spawn(20040)─┬─wsh(20041)
	            │                    ├─{iomux-spawn}(20042)
	            │                    ├─{iomux-spawn}(20043)
	            │                    ├─{iomux-spawn}(20044)
	            │                    ├─{iomux-spawn}(20045)
	            │                    └─{iomux-spawn}(20046)
	            ├─oom(15194)
	            ├─oom(19936)
	            ├─{ruby}(14003)
	            ├─{ruby}(14319)
	            ├─{ruby}(14320)
	            ├─{ruby}(14321)
	            ├─{ruby}(14322)
	            ├─{ruby}(14323)
	            ├─{ruby}(14324)
	            ├─{ruby}(14325)
	            ├─{ruby}(14326)
	            ├─{ruby}(14327)
	            ├─{ruby}(14328)
	            ├─{ruby}(14329)
	            ├─{ruby}(14330)
	            ├─{ruby}(14331)
	            ├─{ruby}(14332)
	            ├─{ruby}(14333)
	            ├─{ruby}(14334)
	            ├─{ruby}(14335)
	            ├─{ruby}(14336)
	            ├─{ruby}(14337)
	            └─{ruby}(14338)
	
可以看到又启动了一套iomux-link、iomux-spawn、oom进程，因此可以确定这些进程是针对一个应用实例启动的。

### warden容器的进程间通信
	
#### 进程打开文件情况
	
通过进程打开的文件描述符可以了解一些进程间通信情况，在此分别查看一下iomux-spawn、iomux-link、oom、wshd、wsh等进程的相应情况。	主要查看类型为FIFO和unix的文件描述符。

iomux-spawn进程打开文件情况：

	paas@ubuntu:~/paas-release.git/bin$ sudo lsof -p 15282
	COMMAND     PID USER   FD   TYPE             DEVICE SIZE/OFF    NODE NAME
	iomux-spa 15282 root  cwd    DIR                8,1     4096  156481 /home/paas/build_dir/target/warden_2fd4fcd33.git/warden
	iomux-spa 15282 root  rtd    DIR                8,1     4096       2 /
	iomux-spa 15282 root  txt    REG                8,1    62561  158449 /home/paas/warden/container/178l4d6v637/bin/iomux-spawn
	iomux-spa 15282 root  mem    REG                8,1  1811128  795078 /lib/x86_64-linux-gnu/libc-2.15.so
	iomux-spa 15282 root  mem    REG                8,1   135366  795080 /lib/x86_64-linux-gnu/libpthread-2.15.so
	iomux-spa 15282 root  mem    REG                8,1   149280  795092 /lib/x86_64-linux-gnu/ld-2.15.so
	iomux-spa 15282 root    0r  FIFO                0,8      0t0 1698219 pipe
	iomux-spa 15282 root    1w  FIFO                0,8      0t0 1698220 pipe
	iomux-spa 15282 root    2w  FIFO                0,8      0t0 1698221 pipe
	iomux-spa 15282 root    3u  unix 0xffff88003a7bf400      0t0 1698241 /home/paas/warden/container/178l4d6v637/jobs/6/stdout.sock
	iomux-spa 15282 root    4u  unix 0xffff88003a7bea40      0t0 1698242 /home/paas/warden/container/178l4d6v637/jobs/6/stderr.sock
	iomux-spa 15282 root    5u  unix 0xffff88003a7bc000      0t0 1698243 /home/paas/warden/container/178l4d6v637/jobs/6/status.sock
	iomux-spa 15282 root    6r  FIFO                0,8      0t0 1698244 pipe
	iomux-spa 15282 root    7r  FIFO                0,8      0t0 1698247 pipe
	iomux-spa 15282 root    8r  FIFO                0,8      0t0 1698245 pipe
	iomux-spa 15282 root    9w  FIFO                0,8      0t0 1698247 pipe
	iomux-spa 15282 root   10r  FIFO                0,8      0t0 1698246 pipe
	iomux-spa 15282 root   11w  FIFO                0,8      0t0 1698246 pipe
	iomux-spa 15282 root   12r  FIFO                0,8      0t0 1698248 pipe
	iomux-spa 15282 root   13w  FIFO                0,8      0t0 1698248 pipe
	iomux-spa 15282 root   14r  FIFO                0,8      0t0 1698249 pipe
	iomux-spa 15282 root   15w  FIFO                0,8      0t0 1698249 pipe
	iomux-spa 15282 root   16r  FIFO                0,8      0t0 1698250 pipe
	iomux-spa 15282 root   17w  FIFO                0,8      0t0 1698250 pipe
	iomux-spa 15282 root   18r  FIFO                0,8      0t0 1698251 pipe
	iomux-spa 15282 root   19w  FIFO                0,8      0t0 1698251 pipe
	iomux-spa 15282 root   20u  unix 0xffff8800237b1040      0t0 1698275 /home/paas/warden/container/178l4d6v637/jobs/6/stdout.sock
	iomux-spa 15282 root   21u  unix 0xffff88003d691040      0t0 1698277 /home/paas/warden/container/178l4d6v637/jobs/6/stderr.sock
	iomux-spa 15282 root   22u  unix 0xffff88003d693a80      0t0 1698279 /home/paas/warden/container/178l4d6v637/jobs/6/status.sock


iomux-link进程打开文件情况：

	paas@ubuntu:~/paas-release.git/bin$ sudo lsof -p 15289
	COMMAND     PID USER   FD   TYPE             DEVICE SIZE/OFF    NODE NAME
	iomux-lin 15289 root  cwd    DIR                8,1     4096  156481 /home/paas/build_dir/target/warden_2fd4fcd33.git/warden
	iomux-lin 15289 root  rtd    DIR                8,1     4096       2 /
	iomux-lin 15289 root  txt    REG                8,1    38948  158448 /home/paas/warden/container/178l4d6v637/bin/iomux-link
	iomux-lin 15289 root  mem    REG                8,1  1811128  795078 /lib/x86_64-linux-gnu/libc-2.15.so
	iomux-lin 15289 root  mem    REG                8,1   135366  795080 /lib/x86_64-linux-gnu/libpthread-2.15.so
	iomux-lin 15289 root  mem    REG                8,1   149280  795092 /lib/x86_64-linux-gnu/ld-2.15.so
	iomux-lin 15289 root    0r  FIFO                0,8      0t0 1698252 pipe
	iomux-lin 15289 root    1w  FIFO                0,8      0t0 1698253 pipe
	iomux-lin 15289 root    2w  FIFO                0,8      0t0 1698254 pipe
	iomux-lin 15289 root    3u  unix 0xffff8800237b2700      0t0 1698274 socket
	iomux-lin 15289 root    4u  unix 0xffff88003d6923c0      0t0 1698276 socket
	iomux-lin 15289 root    5u  unix 0xffff88003d691380      0t0 1698278 socket

oom进程打开文件情况：

	paas@ubuntu:~/paas-release.git/bin$ sudo lsof -p 15194
	COMMAND   PID USER   FD   TYPE DEVICE SIZE/OFF    NODE NAME
	oom     15194 root  cwd    DIR    8,1     4096  156481 /home/paas/build_dir/target/warden_2fd4fcd33.git/warden
	oom     15194 root  rtd    DIR    8,1     4096       2 /
	oom     15194 root  txt    REG    8,1    16959  156582 /home/paas/build_dir/target/warden_2fd4fcd33.git/warden/src/oom/oom
	oom     15194 root  mem    REG    8,1  1811128  795078 /lib/x86_64-linux-gnu/libc-2.15.so
	oom     15194 root  mem    REG    8,1   149280  795092 /lib/x86_64-linux-gnu/ld-2.15.so
	oom     15194 root    0r  FIFO    0,8      0t0 1680292 pipe
	oom     15194 root    1w  FIFO    0,8      0t0 1680293 pipe
	oom     15194 root    2w  FIFO    0,8      0t0 1680294 pipe
	oom     15194 root    3u  0000    0,9        0    4831 anon_inode
	oom     15194 root    4r   REG   0,23        0 1680029 /tmp/warden/cgroup/memory/instance-178l4d6v637/memory.oom_control
	oom     15194 root    5w   REG   0,23        0 1680017 /tmp/warden/cgroup/memory/instance-178l4d6v637/cgroup.event_control

wshd进程打开文件情况：
	
	paas@ubuntu:~/paas-release.git/bin$ sudo lsof -p 15141
	COMMAND   PID USER   FD   TYPE             DEVICE SIZE/OFF    NODE NAME
	wshd    15141 root  cwd    DIR               0,24     4096 1679954 /
	wshd    15141 root  rtd    DIR               0,24     4096 1679954 /
	wshd    15141 root  txt    REG                8,1   972698  158558 /sbin/wshd
	wshd    15141 root    0u  0000                0,9        0    4831 anon_inode
	wshd    15141 root    3u  unix 0xffff88003a7bdd40      0t0 1679949 ./run/wshd.sock
	wshd    15141 root   11w  FIFO                0,8      0t0 1698315 pipe	
wsh进程打开文件情况：

	paas@ubuntu:~/paas-release.git/bin$ sudo lsof -p 15283
	COMMAND   PID USER   FD   TYPE             DEVICE SIZE/OFF    NODE NAME
	wsh     15283 root  cwd    DIR                8,1     4096  156481 /home/paas/build_dir/target/warden_2fd4fcd33.git/warden
	wsh     15283 root  rtd    DIR                8,1     4096       2 /
	wsh     15283 root  txt    REG                8,1   945289  158447 /home/paas/warden/container/178l4d6v637/bin/wsh
	wsh     15283 root    1w  FIFO                0,8      0t0 1698244 pipe
	wsh     15283 root    2w  FIFO                0,8      0t0 1698245 pipe
	wsh     15283 root    3u  unix 0xffff88003d690000      0t0 1698310 socket
	wsh     15283 root    5r  FIFO                0,8      0t0 1698313 pipe
	wsh     15283 root    6r  FIFO                0,8      0t0 1698314 pipe
	wsh     15283 root    7r  FIFO                0,8      0t0 1698315 pipe
	
wshd进程的子进程bash进程打开文件情况：

	paas@ubuntu:~/paas-release.git/bin$ sudo lsof -p 15292
	lsof: no pwd entry for UID 10001
	lsof: no pwd entry for UID 10001
	lsof: no pwd entry for UID 10001
	lsof: no pwd entry for UID 10001
	lsof: no pwd entry for UID 10001
	lsof: no pwd entry for UID 10001
	lsof: no pwd entry for UID 10001
	lsof: no pwd entry for UID 10001
	lsof: no pwd entry for UID 10001
	lsof: no pwd entry for UID 10001
	lsof: no pwd entry for UID 10001
	lsof: no pwd entry for UID 10001
	lsof: no pwd entry for UID 10001
	lsof: no pwd entry for UID 10001
	COMMAND   PID     USER   FD   TYPE DEVICE SIZE/OFF    NODE NAME
	lsof: no pwd entry for UID 10001
	bash    15292    10001  cwd    DIR   0,24     4096 1680395 /home/vcap
	lsof: no pwd entry for UID 10001
	bash    15292    10001  rtd    DIR   0,24     4096 1679954 /
	lsof: no pwd entry for UID 10001
	bash    15292    10001  txt    REG    8,1   959120  810540 /bin/bash
	lsof: no pwd entry for UID 10001
	bash    15292    10001  mem    REG    8,1           810263 /lib/x86_64-linux-gnu/libnss_files-2.15.so (path inode=795088)
	lsof: no pwd entry for UID 10001
	bash    15292    10001  mem    REG    8,1           810277 /lib/x86_64-linux-gnu/libnss_nis-2.15.so (path inode=795084)
	lsof: no pwd entry for UID 10001
	bash    15292    10001  mem    REG    8,1           810203 /lib/x86_64-linux-gnu/libnsl-2.15.so (path inode=795097)
	lsof: no pwd entry for UID 10001
	bash    15292    10001  mem    REG    8,1           810216 /lib/x86_64-linux-gnu/libnss_compat-2.15.so (path inode=795089)
	lsof: no pwd entry for UID 10001
	bash    15292    10001  mem    REG    8,1           810200 /lib/x86_64-linux-gnu/libc-2.15.so (path inode=795078)
	lsof: no pwd entry for UID 10001
	bash    15292    10001  mem    REG    8,1           810243 /lib/x86_64-linux-gnu/libdl-2.15.so (path inode=786457)
	lsof: no pwd entry for UID 10001
	bash    15292    10001  mem    REG    8,1           810285 /lib/x86_64-linux-gnu/libtinfo.so.5.9 (path 	inode=786531)
	lsof: no pwd entry for UID 10001
	bash    15292    10001  mem    REG    8,1           810142 /lib/x86_64-linux-gnu/ld-2.15.so (path inode=795092)
	lsof: no pwd entry for UID 10001
	bash    15292    10001    0r  FIFO    0,8      0t0 1698312 pipe
	lsof: no pwd entry for UID 10001
	bash    15292    10001    1w  FIFO    0,8      0t0 1698313 pipe
	lsof: no pwd entry for UID 10001
	bash    15292    10001    2w  FIFO    0,8      0t0 1698314 pipe
	
wshd进程的子进程bash进程的子进程startup进程打开文件情况：

	paas@ubuntu:~/paas-release.git/bin$ sudo lsof -p 15304
	lsof: no pwd entry for UID 10001
	lsof: no pwd entry for UID 10001
	lsof: no pwd entry for UID 10001
	lsof: no pwd entry for UID 10001
	lsof: no pwd entry for UID 10001
	lsof: no pwd entry for UID 10001
	lsof: no pwd entry for UID 10001
	lsof: no pwd entry for UID 10001
	lsof: no pwd entry for UID 10001
	lsof: no pwd entry for UID 10001
	lsof: no pwd entry for UID 10001
	lsof: no pwd entry for UID 10001
	lsof: no pwd entry for UID 10001
	lsof: no pwd entry for UID 10001
	lsof: no pwd entry for UID 10001
	lsof: no pwd entry for UID 10001
	lsof: no pwd entry for UID 10001
	COMMAND   PID     USER   FD   TYPE DEVICE SIZE/OFF    NODE NAME
	lsof: no pwd entry for UID 10001
	startup 15304    10001  cwd    DIR   0,24     4096 1680396 /home/vcap/app
	lsof: no pwd entry for UID 10001
	startup 15304    10001  rtd    DIR   0,24     4096 1679954 /
	lsof: no pwd entry for UID 10001
	startup 15304    10001  txt    REG    8,1   959120  810540 /bin/bash
	lsof: no pwd entry for UID 10001
	startup 15304    10001  mem    REG    8,1           810263 /lib/x86_64-linux-gnu/libnss_files-2.15.so (path inode=795088)
	lsof: no pwd entry for UID 10001
	startup 15304    10001  mem    REG    8,1           810277 /lib/x86_64-linux-gnu/libnss_nis-2.15.so (path inode=795084)
	lsof: no pwd entry for UID 10001
	startup 15304    10001  mem    REG    8,1           810203 /lib/x86_64-linux-gnu/libnsl-2.15.so (path inode=795097)
	lsof: no pwd entry for UID 10001
	startup 15304    10001  mem    REG    8,1           810216 /lib/x86_64-linux-gnu/libnss_compat-2.15.so (path inode=795089)
	lsof: no pwd entry for UID 10001
	startup 15304    10001  mem    REG    8,1           810200 /lib/x86_64-linux-gnu/libc-2.15.so (path inode=795078)
	lsof: no pwd entry for UID 10001
	startup 15304    10001  mem    REG    8,1           810243 /lib/x86_64-linux-gnu/libdl-2.15.so (path inode=786457)
	lsof: no pwd entry for UID 10001
	startup 15304    10001  mem    REG    8,1           810285 /lib/x86_64-linux-gnu/libtinfo.so.5.9 (path inode=786531)
	lsof: no pwd entry for UID 10001
	startup 15304    10001  mem    REG    8,1           810142 /lib/x86_64-linux-gnu/ld-2.15.so (path inode=795092)
	lsof: no pwd entry for UID 10001
	startup 15304    10001  mem    REG    8,1           147791 /usr/lib/x86_64-linux-gnu/gconv/gconv-modules.cache (path inode=410628)
	lsof: no pwd entry for UID 10001
	startup 15304    10001  mem    REG    8,1           802495 /usr/lib/locale/locale-archive (path inode=400796)
	lsof: no pwd entry for UID 10001
	startup 15304    10001    0r  FIFO    0,8      0t0 1698312 pipe
	lsof: no pwd entry for UID 10001
	startup 15304    10001    1w  FIFO    0,8      0t0 1698313 pipe
	lsof: no pwd entry for UID 10001
	startup 15304    10001    2w  FIFO    0,8      0t0 1698314 pipe
	lsof: no pwd entry for UID 10001
	startup 15304    10001  255r   REG    8,1      510  158576 /home/vcap/startup	
	
未完待续...

[Linux中的namespaces]: http://lsword.github.io/2013/09/20.html