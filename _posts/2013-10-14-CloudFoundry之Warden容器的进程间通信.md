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
	
未完待续...

[Linux中的namespaces]: http://lsword.github.io/2013/09/20.html