---
layout: post
category: 云计算
tags: docker
description: 本文记录docker官方的镜像仓库docker-registry的安装、使用过程，以及对其文件组织的分析。
---

### 背景

本文中涉及的源码、测试环境都是基于docker 0.11.1版本系统和docker-registry 0.6.5版本系统。

docker-registry是docker官方提供的镜像仓库，可以用于构建私有的镜像仓库。

### 安装

#### 获取代码

	git clone https://github.com/dotcloud/docker-registry.git
	
#### 修改配置
	cat docker-registry/config/config.yml
	
	common:
	    # Bucket for storage
    	boto_bucket: REPLACEME

	    search_backend: "_env:SEARCH_BACKEND:"
    	sqlalchemy_index_database:
        	"_env:SQLALCHEMY_INDEX_DATABASE:sqlite:////tmp/docker-registry.db"

	    # Let gunicorn set this environment variable or set a random string here
    	secret_key: _env:SECRET_KEY
	# The `common' part is automatically included (and possibly overriden by all
	# other flavors)
	dev:
    	storage: local
	    storage_path: /paas-disk/docker/registry
    	loglevel: debug
    
	以上配置使用本地存储。
	
#### 启动准备

	安装相关软件
	sudo apt-get install build-essential python-dev libevent-dev python-pip libssl-dev liblzma-dev libffi-dev
	
	安装应用
	sudo pip install .
	
	以上为ubuntu环境的启动准备。

#### 启动

	gunicorn --access-logfile - --debug -k gevent -b 0.0.0.0:5000 -w 1 docker_registry.wsgi:application &
	
### 使用

#### 注册

#### 获取镜像

#### 上传镜像

### 文件组织

### 其他

[官方文档]

[官方文档]: http://docs.docker.io/use/working_with_links_names