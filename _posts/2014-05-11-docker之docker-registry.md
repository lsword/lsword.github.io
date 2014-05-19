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

#### 注册&登陆

	docker客户端默认的docker-registry地址是https://index.docker.io，如果需要连接到私有的docker-registry，需要使用docker login命令指定私有的docker-registry。
	docker login命令会要求输入用户名、密码、email，这个目前可以随便输入。

#### 获取镜像

	docker pull 10.1.1.64:5000/centos:6.4
	命令中需要指定私有docker-registry的地址。
	
#### 上传镜像

	docker tag 539c0211cd76 10.1.1.64:5000/centos:6.4
	docker tag b48b681ac984 10.1.1.64:5000/centos:6.5
	docker tag b48b681ac984 10.1.1.64:5000/centos:latest
	docker push 10.1.1.64:5000/centos
	
	
### 文件组织

以下是一个测试环境中docker-registry保存docker镜像的目录结构信息。

~~~

paas@ubuntu:/paas-disk/docker/registry$ tree
.
├── images
│   ├── 02dae1c13f51edae0a9817e01dcbbad380c1b31933779641e5f733958af5d8d5
│   │   ├── ancestry
│   │   ├── _checksum
│   │   ├── json
│   │   └── layer
│   ├── 07302703beccc2ea25f34333decad32ed06446e8a14c020ffbd0be017364b9fe
│   │   ├── ancestry
│   │   ├── _checksum
│   │   ├── json
│   │   └── layer
│   ├── 0b443ba0395813ef287c27f5ff953121a69ab23c467ffbefcefaec3f255e0693
│   │   ├── ancestry
│   │   ├── _checksum
│   │   ├── json
│   │   └── layer
│   ├── 2209cbf9dcd35615211a2fdc6762bb5e651b5c847537359f05b9ab1bc9a74614
│   │   ├── ancestry
│   │   ├── _checksum
│   │   ├── json
│   │   └── layer
│   ├── 316b678ddf487a37012630ae3219c8bb78c1f4b58d31c9513c3ea6b88f9e5635
│   │   ├── ancestry
│   │   ├── _checksum
│   │   ├── json
│   │   └── layer
│   ├── 4d26dd3ebc1c823cfa652280eca0230ec411fb6a742983803e49e051fe367efe
│   │   ├── ancestry
│   │   ├── _checksum
│   │   ├── json
│   │   └── layer
│   ├── 511136ea3c5a64f264b78b5433614aec563103b4d4702f3ba7d4d2698e22c158
│   │   ├── ancestry
│   │   ├── _checksum
│   │   ├── json
│   │   └── layer
│   ├── 514b348987eff7991ad24b3fa26838f9768f41a618a1f78c8a4e6d5382b14df9
│   │   ├── ancestry
│   │   ├── _checksum
│   │   ├── json
│   │   └── layer
│   ├── 539c0211cd76cdeaedbecf9f023ef774612e331137ce7ebe4ae1b61088e7edbe
│   │   ├── ancestry
│   │   ├── _checksum
│   │   ├── json
│   │   └── layer
│   ├── 5e019ab7bf6deb75b211411ef7257d1e76bf7edee31d9da62a392df98d0529d6
│   │   ├── ancestry
│   │   ├── _checksum
│   │   ├── json
│   │   └── layer
│   ├── 5e66087f3ffe002664507d225d07b6929843c3f0299f5335a70c1727c8833737
│   │   ├── ancestry
│   │   ├── _checksum
│   │   ├── json
│   │   └── layer
│   ├── 7064731afe90d78da2c117d64a1221c826234cd7145fd330ae7e207ff5606980
│   │   ├── ancestry
│   │   ├── _checksum
│   │   ├── json
│   │   └── layer
│   ├── 7c0d7c261905a9fe83d1aea4037a265058f47e1e3e1134daac879393741cc016
│   │   ├── ancestry
│   │   ├── _checksum
│   │   ├── json
│   │   └── layer
│   ├── 8aedcdadf14bb95a53f712aace7d0cb657447d24e41831518392d1a2b05d8bf7
│   │   ├── ancestry
│   │   ├── _checksum
│   │   ├── json
│   │   └── layer
│   ├── 99ec81b80c55d906afd8179560fdab0ee93e32c52053816ca1d531597c1ff48f
│   │   ├── ancestry
│   │   ├── _checksum
│   │   ├── json
│   │   └── layer
│   ├── a7cf8ae4e998c5339e769d6cc466f9133bd4d330a549bb846cb1641cd638247c
│   │   ├── ancestry
│   │   ├── _checksum
│   │   ├── json
│   │   └── layer
│   ├── cb12405ee8fa58aa11f2a2fe5ab98430fefdf330ce1a9bd37ba800abc14b9ca1
│   │   ├── ancestry
│   │   ├── _checksum
│   │   ├── json
│   │   └── layer
│   ├── cf8dc907452c970224551599da573c9e32897fc65286d942625c4c86dabd680d
│   │   ├── ancestry
│   │   ├── _checksum
│   │   ├── json
│   │   └── layer
│   ├── d4010efcfd86c7f59f6b83b90e9c66d4cc4d78cd2266e853b95d464ea0eb73e6
│   │   ├── ancestry
│   │   ├── _checksum
│   │   ├── json
│   │   └── layer
│   ├── e2aa6665d37109cd6d196eec32c98d8f4412325cb2dc1588e5eb27cae4beb836
│   │   ├── ancestry
│   │   ├── _checksum
│   │   ├── json
│   │   └── layer
│   ├── e7206bfc66aac0ab946eb11dad1271f4f5f3f72ab65cff66b4092aeaee20475b
│   │   ├── ancestry
│   │   ├── _checksum
│   │   ├── json
│   │   └── layer
│   ├── ef519c9ee91a06fc33cefbda1bce27686617761700252dff0397f2c0e269f3c5
│   │   ├── ancestry
│   │   ├── _checksum
│   │   ├── json
│   │   └── layer
│   ├── f030997216f3f605c3db2e60fac9a327910f30319b3fae95a457f87801bc2734
│   │   ├── ancestry
│   │   ├── _checksum
│   │   ├── json
│   │   └── layer
│   └── f0ee64c4df74c766a16faebfb5123e2b67192eff4a82aa70d0d2081681200a79
│       ├── ancestry
│       ├── _checksum
│       ├── json
│       └── layer
└── repositories
    └── library
        ├── centos
        │   ├── _index_images
        │   ├── json
        │   ├── tag_6.4
        │   ├── tag6.4_json
        │   ├── tag_6.5
        │   ├── tag6.5_json
        │   ├── tag_latest
        │   └── taglatest_json
        └── ubuntu
            ├── _index_images
            ├── json
            ├── tag_12.10
            ├── tag12.10_json
            ├── tag_13.04
            ├── tag13.04_json
            ├── tag_13.10
            ├── tag13.10_json
            ├── tag_14.04
            ├── tag14.04_json
            ├── tag_latest
            ├── taglatest_json
            ├── tag_quantal
            ├── tagquantal_json
            ├── tag_raring
            ├── tagraring_json
            ├── tag_saucy
            ├── tagsaucy_json
            ├── tag_trusty
            └── tagtrusty_json
            
~~~            
            
从上面可以看出文件主要分为两部分组织，一部分是实际镜像相关文件，保存在images目录，一部分是仓库描述文件，保存在repositories目录。

#### repositories

不同名称的镜像保存在单独的目录，可以看到centos和ubuntu分不同目录存储。

每个镜像的每个tag由两个文件描述，例如tag_14.04和tag14.04_json

	cat tag14.04_json
	{"kernel": "3.13.0-24-generic", "arch": "amd64", "docker_go_version": "go1.2.1", "last_update": 1400116767, "docker_version": "0.11.1", "os": "linux"}
	
	cat tag_14.04
	8aedcdadf14bb95a53f712aace7d0cb657447d24e41831518392d1a2b05d8bf7
	cat tag_trusty
	8aedcdadf14bb95a53f712aace7d0cb657447d24e41831518392d1a2b05d8bf7
	cat tag_latest
	8aedcdadf14bb95a53f712aace7d0cb657447d24e41831518392d1a2b05d8bf7
	
	可以看到，tag_14.04记录的是ubuntu:14.04镜像的id。tag_14.04、tag_trusty、tag_latest对应同一个镜像。
	
	cat tag_12.10
	f030997216f3f605c3db2e60fac9a327910f30319b3fae95a457f87801bc2734
	cat tag_13.04
	514b348987eff7991ad24b3fa26838f9768f41a618a1f78c8a4e6d5382b14df9
	cat tag_13.10
	7c0d7c261905a9fe83d1aea4037a265058f47e1e3e1134daac879393741cc016
	cat tag_14.04
	8aedcdadf14bb95a53f712aace7d0cb657447d24e41831518392d1a2b05d8bf7

	

_index_images文件内容如下：

~~~     
cat _index_images
[{"id": "e7206bfc66aac0ab946eb11dad1271f4f5f3f72ab65cff66b4092aeaee20475b"}, {"id": "d4010efcfd86c7f59f6b83b90e9c66d4cc4d78cd2266e853b95d464ea0eb73e6"}, {"id": "511136ea3c5a64f264b78b5433614aec563103b4d4702f3ba7d4d2698e22c158"}, {"id": "cb12405ee8fa58aa11f2a2fe5ab98430fefdf330ce1a9bd37ba800abc14b9ca1"}, {"id": "2209cbf9dcd35615211a2fdc6762bb5e651b5c847537359f05b9ab1bc9a74614"}, {"id": "cf8dc907452c970224551599da573c9e32897fc65286d942625c4c86dabd680d"}, {"id": "02dae1c13f51edae0a9817e01dcbbad380c1b31933779641e5f733958af5d8d5"}, {"id": "316b678ddf487a37012630ae3219c8bb78c1f4b58d31c9513c3ea6b88f9e5635"}, {"id": "7c0d7c261905a9fe83d1aea4037a265058f47e1e3e1134daac879393741cc016"}, {"id": "8aedcdadf14bb95a53f712aace7d0cb657447d24e41831518392d1a2b05d8bf7"}, {"id": "99ec81b80c55d906afd8179560fdab0ee93e32c52053816ca1d531597c1ff48f"}, {"id": "07302703beccc2ea25f34333decad32ed06446e8a14c020ffbd0be017364b9fe"}, {"id": "4d26dd3ebc1c823cfa652280eca0230ec411fb6a742983803e49e051fe367efe"}, {"id": "e2aa6665d37109cd6d196eec32c98d8f4412325cb2dc1588e5eb27cae4beb836"}, {"id": "5e66087f3ffe002664507d225d07b6929843c3f0299f5335a70c1727c8833737"}, {"id": "5e019ab7bf6deb75b211411ef7257d1e76bf7edee31d9da62a392df98d0529d6"}, {"id": "ef519c9ee91a06fc33cefbda1bce27686617761700252dff0397f2c0e269f3c5"}, {"id": "f0ee64c4df74c766a16faebfb5123e2b67192eff4a82aa70d0d2081681200a79"}, {"id": "514b348987eff7991ad24b3fa26838f9768f41a618a1f78c8a4e6d5382b14df9"}, {"id": "a7cf8ae4e998c5339e769d6cc466f9133bd4d330a549bb846cb1641cd638247c"}, {"id": "f030997216f3f605c3db2e60fac9a327910f30319b3fae95a457f87801bc2734"}]
~~~     

可以看到，这个目录中只有四个版本的ubuntu镜像，但是_index_images文件中有21个id号。其余的17个id实际上是最终的四个版本的ubuntu镜像的各级祖先镜像的id。从这个目录中的文件中是无法看出镜像的继承关系的需要查看images目录中的相关文件。

#### images

此目录中保存各个镜像的数据。每个镜像一个目录，每个目录中有4个文件：

	ancestry：此文件记录当前镜像的继承关系
	_checksum：校验码
	json：镜像描述信息
	layer：本层的镜像文件

##### ancestry

以ubuntu14.04和ubuntu13.10为例：

	cat ancestry
	["8aedcdadf14bb95a53f712aace7d0cb657447d24e41831518392d1a2b05d8bf7", "99ec81b80c55d906afd8179560fdab0ee93e32c52053816ca1d531597c1ff48f", "d4010efcfd86c7f59f6b83b90e9c66d4cc4d78cd2266e853b95d464ea0eb73e6", "4d26dd3ebc1c823cfa652280eca0230ec411fb6a742983803e49e051fe367efe", "5e66087f3ffe002664507d225d07b6929843c3f0299f5335a70c1727c8833737", "511136ea3c5a64f264b78b5433614aec563103b4d4702f3ba7d4d2698e22c158"]
	可以看到ubuntu14.04的镜像是由6个镜像叠加而成的。
	下面再看一下ubuntu13.10的ancestry文件：
	cat ancestry
	["7c0d7c261905a9fe83d1aea4037a265058f47e1e3e1134daac879393741cc016", "5e019ab7bf6deb75b211411ef7257d1e76bf7edee31d9da62a392df98d0529d6", "2209cbf9dcd35615211a2fdc6762bb5e651b5c847537359f05b9ab1bc9a74614", "f0ee64c4df74c766a16faebfb5123e2b67192eff4a82aa70d0d2081681200a79", "e2aa6665d37109cd6d196eec32c98d8f4412325cb2dc1588e5eb27cae4beb836", "511136ea3c5a64f264b78b5433614aec563103b4d4702f3ba7d4d2698e22c158"]
	可以看到ubuntu13.10的镜像也是是由6个镜像叠加而成的。
	并且ubuntu13.10和ubuntu14.04有共同的祖先511136ea3c5a64f264b78b5433614aec563103b4d4702f3ba7d4d2698e22c158。

通过查看4个版本的ubuntu的image的ancestry文件可以发现，它们都有共同的祖先511136ea3c5a64f264b78b5433614aec563103b4d4702f3ba7d4d2698e22c158，都由6个image组成，因此可以解释_index_images文件中为什么会有21个id号（6*4-3）。

以centos6.5为例：

	cat ancestry
	["0b443ba0395813ef287c27f5ff953121a69ab23c467ffbefcefaec3f255e0693", "7064731afe90d78da2c117d64a1221c826234cd7145fd330ae7e207ff5606980", "511136ea3c5a64f264b78b5433614aec563103b4d4702f3ba7d4d2698e22c158"]
	可以看到centos6.5的镜像是由3个镜像叠加而成的。
	centos6.5的镜像与ubuntu14.04有共同的祖先511136ea3c5a64f264b78b5433614aec563103b4d4702f3ba7d4d2698e22c158。

##### json

用于描述image的信息。

以ubuntu14.04为例：

~~~
{
"id":"8aedcdadf14bb95a53f712aace7d0cb657447d24e41831518392d1a2b05d8bf7",
"parent":"99ec81b80c55d906afd8179560fdab0ee93e32c52053816ca1d531597c1ff48f",
"created":"2014-05-15T02:37:51.21802123Z",
"container":"1d4f82ac317735ab5d51853fc5d61346351b79fb41e9e961172e45f0dba31e75",
"container_config"{
	"Hostname":"1d4f82ac3177",
	"Domainname":"",
	"User"    :"",
	"Memory":0,
	"MemorySwap":0,
	"CpuShares"0,
	"AttachStdin":true,
	"AttachStdout":true,
	"AttachStderr":true,
	"PortSpecs":null,
	"ExposedPorts":null,
	"Tty":true,
	"OpenStdin":true,"    
	StdinOnce":true,
	"Env":[
		"HOME=/",
		"PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"],
	"Cmd":["/bin/bash"],
	"Image":"99ec81b80c55",
	"Volumes":null,
	"WorkingDir":"",
	"Entrypoint":null,
	"NetworkDisabled":false,
	"OnBuild":null
},
"docker_version":"0.10.0",
"config":{
	"Hostname":"",
	"Domainname":"",
	"User":"",
	"Memory":0,
	"MemorySwap":0,
	"CpuShares"0,
	"AttachStdin":false,
	"AttachStdout":false,
	"AttachStderr":false,
	"PortSpecs":null,
	"ExposedPorts":null,
	"Tty":true,
	"OpenStdin":true,
	"StdinOnce":true,
	"Env":[
		"HOME=/",
		"PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"],
	"Cmd":["/bin/bash"],
	"Image":"",
	"Volumes":null,
	"WorkingDir":"",
	"Entrypoint":null,
	"NetworkDisabled":false,
	"OnBuild":null
},
"architecture":"amd64",
"os":"linux",
"Size":43197746}
~~~

##### layer

	layer实际上就是一个tgz格式的文件，可以使用tar xfz来解压。
	
	共同祖先511136ea3c5a64f264b78b5433614aec563103b4d4702f3ba7d4d2698e22c158的layer文件为86个字节。其中保存的是对空文件夹的压缩。
	
### 其他

[官方文档]

[官方文档]: http://docs.docker.io/use/working_with_links_names