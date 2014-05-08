---
layout: post
category: Linux
tags: 网络
description: socat是一个可以建立双向字节流并可以在其间传送数据的工具，是在学习docker容器跨主机通信时发现的。本文主要用于记录一些从网上收集到的使用方法。
---

### 文件传送

服务端

	socat -u open:filename,binary tcp-listen:12345
	
客户端

	socat -u tcp:serverip:12345 open:localfilename,create,binary
	
说明

	-u 表示数据单向传送，从第一个参数传递到第二个参数，如果是-U则表示从第二个参数传送到第一个参数。
	open 表示使用系统调用open()打开文件，不能打开unix域socket。
	binary 表示以二进制方式打开文件。以文本方式打开使用text。
	tcp-listen 表示监听tcp端口。
	create 表示如果文件不存在则创建。
	传输结束后两端均退出。

### 读写分离

socat支持打开端的读写分离，使用!!符号，左侧表示读，右侧表示写。

	socat open:hello.html\!\!open:log.txt,create,append tcp-listen:12345,reuseaddr,fork
	
说明

	open:hello.html 表示读hello.html文件。
	open:log.txt 表示收到的数据写入log.txt文件。
	reuseaddr 见socket的SO_REUSEADDR。
	fork 请求到达时，fork一个进程进行处理。
	在bash下，需要用\对!进行转义。
	
### shell

服务端

	socat tcp-l:12345 system:bash,pty,stderr
	
客户端

	socat - tcp:serverip:12345
	
说明

	tcp-l: 同tcp-listen
	system:bash,pty,stderr: 执行bash，标准输出重定向到标准错误输出
	问题: ubuntu下，按照官方文档，客户端执行socat readline tcp:serverip:12345，出现错误提示：socat[2067] E unknown device/address "READLINE"，本机上已经安装readline相关库。

### 其他

还有很多用法后续逐步补充吧。

[socat官方文档]
	
[socat官方文档]: http://www.dest-unreach.org/socat/doc/socat.html