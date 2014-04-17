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

deaagent根据dea模块的instances.json文件获取dea的应用实例信息。从应用实例信息中可以获取到应用标准输出和应用标准错误输出的unix域socket文件的路径，deaagent可以连接到相应的域socket，从中读取应用标准输出和应用标准错误输出的信息。（应用标准输出和应用标准错误输出的信息如何能通过相应的域socket读取，会在介绍warden的相关文章中说明。）

读取到信息后，deaagent通过loggregatorlib中的emitter模块将信息发送到loggregator模块进行处理。

### loggregator


### traffic_controller

	
