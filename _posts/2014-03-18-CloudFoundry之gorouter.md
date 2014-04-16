---
layout: post
category: 云计算
tags: CloudFoundry
description: gorouter是CloudFoundry中用于处理用户请求的入口代理模块，本文主要介绍如何gorouter的主要功能和部分实现细节。
---

### 主要功能
  1. 路由维护
  2. 负载均衡
  3. 会话保持
  4. websocket
  4. 访问日志

### 主要交互对象

#### 开发者

开发者使用客户端工具登陆、上传应用、管理应用等请求都是先到gorouter，再由gorouter转发给cc或uaa。

#### 用户

用户使用浏览器访问应用，请求先到gorouter，再由gorouter转发给相应的应用实例。

#### nats

gorouter从nats上接收由cc或dea发布的应用路由更新信息，并对路由表进行相应处理。

### 关键数据结构

#### Endpoint

记录应用id与应用实例id、应用实例主机ip、应用实例端口之间的对应关系。

#### Pool

记录应用实例主机ip、应用实例端口与Endpoint之间的对应关系。

#### Registry

	byUri：记录域名与Pool的对应关系。

	table：记录应用实例的ip:port、域名与Endpoint、上次更新时间之间的对应关系。


### 路由维护

gorouter从nats上接收应用路由更新信息，对Registry进行相应的增删操作。

gorouter定时对失效的应用路由信息进行清理。

### 负载均衡

收到应用访问请求后，gorouter根据访问域名查找对应的应用实例注册信息，如果找到多个，就随机取一个实例来处理访问。（pool.go的Sample函数）
	
### 会话保持

如果应用实例向客户端回复的响应中存在cookies，gorouter会在响应的cookies中加入应用实例的id，cookie的key值为“JSESSIONID”，如果客户端请求中的cookie中有JSESSIONID，gorouter将会取出响应的应用实例id，并找到响应应用实例所在的主机和端口，将请求转发过去。（参考proxy.go中的lookup函数和request_handler.go中的setupStickySession函数）

### websocket

gorouter支持应用使用websocket方式提供服务。

### 访问日志

gorouter的应用访问日志有两种记录方式，一种记录本地文件，一种是转发给loggregrator进程。对于记录本地文件的方式，不支持文件切分操作。

### 其他


	
