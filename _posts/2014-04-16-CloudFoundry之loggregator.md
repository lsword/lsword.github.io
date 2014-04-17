---
layout: post
category: 云计算
tags: CloudFoundry
description: loggregator是CloudFoundry中用于处理应用日志的子系统，本文主要介绍loggregator的主要功能和部分实现细节。
---

### 组成
  1. deaagent
  2. loggregator
  3. traffic_controller
  
### 1 deaagent

#### 1.1 主要功能

deaagent根据dea模块的instances.json文件获取dea的应用实例信息。从应用实例信息中可以获取到应用标准输出和应用标准错误输出的unix域socket文件的路径，deaagent可以连接到相应的域socket，从中读取应用标准输出和应用标准错误输出的信息。（应用标准输出和应用标准错误输出的信息如何能通过相应的域socket读取，会在介绍warden的相关文章中说明。）

读取到信息后，deaagent通过loggregatorlib中的emitter模块将信息发送到loggregator模块进行处理。

#### 1.2 实现细节

##### 1.2.1 instances.json文件监控

使用了fsnotify(https://github.com/howeyc/fsnotify)实现对文件的监控。

如果监控到instances.json文件被删除，则取消所有应用日志信息读取操作。

如果监控到对文件instances.json的其他操作，则重新读取instances.json文件，取消当前所有应用日志信息读取操作，根据当前文件内容重新启动对应用日志信息的读取操作。（不能保证日志全部被取到）

fsnotify这个go语言库还是很有用的，可以用到很多实际问题。

##### 1.2.2 使用channel实现原子操作

###### channel定义

~~~
type agent struct {
	InstancesJsonFilePath string
	logger                *gosteno.Logger
	knownInstancesChan    chan<- func(map[string]*TaskListener)
}
~~~

上面代码中的knownInstancesChan是一个函数类型的channel，这种使用channel的方式头次遇到。

###### channel初始化及执行

~~~
knownInstancesChan := atomicCacheOperator()

func atomicCacheOperator() chan<- func(map[string]*TaskListener) {
	operations := make(chan func(map[string]*TaskListener))
	go func() {
		knownTasks := make(map[string]*TaskListener)
		for operation := range operations {
			operation(knownTasks)
		}
	}()
	return operations
}
~~~

可以看到，只要channel中有函数，就会被取出并执行。

###### channel使用 

~~~
agent.knownInstancesChan <- agent.processTasks(currentTasks, emitter)
~~~

### 2 loggregator

#### 2.1 主要功能

loggregator接收从deaagent、gorouter、cloud_controller_ng等模块发来的日志信息，将信息推送到使用cli终端(基于websocket)或发送到应用绑定的外部日志服务器。

目前只有go语言版的客户端cli(ruby版的用不了loggregator)能够使用loggregator提供的日志推送服务。

#### 2.2 日志接收

loggregator启动udp监听服务来接收外部系统发送的日志信息。

#### 2.3 日志发送

loggregator支持三种日志发送方式。

##### 2.3.1 tail

使用cli获取应用日志时，默认使用tail方式。这种情况下，cli作为websocket客户端，连接上loggregator并等待loggregator推送消息。

##### 2.3.2 dump

使用cli获取应用日志时，加上--recent参数，使用dump方式获取日志，即获取最近的几条日志。

##### 2.3.3 syslog

用户可以使用cli(ruby版不支持)创建一个用户自定义服务，提供一个可以发送syslog日志格式(可参考RFC5424、RFC5425、RFC6587)的服务url地址，loggregator可以将应用的标准输出和标准错误输出发送到相应的url地址。配置过程如下：

~~~
1. 去papertrail或splunkstorm申请一个日志服务，获取日志发送的url地址。
2. gcf cups 服务名 -l syslog://日志发送的url地址。
3. gcf bs 应用名 服务名
4. 重新push应用
~~~

试用过papertrail、loggly、splunk storm、sumo logic、loggr、logentries、keen.io、flydata等云服务网站，好像适用的只有papertrail和splunkstorm。

#### 2.4 实现细节

##### 2.4.1 websocket

使用gorilla/websocket(https://github.com/gorilla/websocket)实现支持，协议是RFC6455。功能上比go官方自带的库强。

##### 2.4.2 syslog

如果要支持syslog方式，需要启动etcd进程。loggregator使用etcd来保存应用绑定的日志服务信息并对其改变进行监控，如果发生变化则进行相应处理。

### 3 traffic_controller

待补充
	
