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
  
### deaagent

#### 主要功能

deaagent根据dea模块的instances.json文件获取dea的应用实例信息。从应用实例信息中可以获取到应用标准输出和应用标准错误输出的unix域socket文件的路径，deaagent可以连接到相应的域socket，从中读取应用标准输出和应用标准错误输出的信息。（应用标准输出和应用标准错误输出的信息如何能通过相应的域socket读取，会在介绍warden的相关文章中说明。）

读取到信息后，deaagent通过loggregatorlib中的emitter模块将信息发送到loggregator模块进行处理。

#### 实现细节

##### 1. instances.json文件监控

使用了fsnotify(https://github.com/howeyc/fsnotify)实现对文件的监控。

如果监控到instances.json文件被删除，则取消所有应用日志信息读取操作。

如果监控到对文件instances.json的其他操作，则重新读取instances.json文件，取消当前所有应用日志信息读取操作，根据当前文件内容重新启动对应用日志信息的读取操作。（不能保证日志全部被取到）

fsnotify这个go语言库还是很有用的，可以用到很多实际问题。

##### 2. 使用channel实现原子操作

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

### loggregator


### traffic_controller

	
