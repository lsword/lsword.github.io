---
layout: post
category: 云计算
tags: mesos,marathon,paas
description: 本文介绍marathon-lb的使用
---

## 一. 安装部署

### 1. 获取镜像

    pull mesosphere/marathon-lb

### 2. 使用docker命令启动

    docker -d -e PORTS=9091 --privileged --net=host docker.io/mesosphere/marathon-lb sse --marathon http://192.168.36.111:8080 --group external

### 3. 使用marathon启动

*   marathon-lb.json

~~~

    {
      "id": "marathon-lb",
      "cmd": "/marathon-lb/run sse --marathon http://192.168.36.111:8080 --group external",
      "cpus": 1,
      "mem": 128,
      "disk": 0,
      "instances": 1,
      "container": {
        "docker": {
          "image": "docker.io/mesosphere/marathon-lb",
          "network": "HOST",
          "privileged": true,
          "parameters": []
        },
      "type": "DOCKER",
      volumes": [],
      "constraints": [
        [
          "hostname",
          "CLUSTER",
          "192.168.36.107"
        ]
      ],
      "portDefinitions": [
        {
          "port": 0,
          "protocol": "tcp",
          "name": null,
          "labels": null
        }
      ],
      "env": {},
      "labels": {}
    }

~~~

*   curl命令

    curl -i -H 127.00.1:8080/v2/apps -d @marathon-lb.json

    ### 4. API Endpoints

    <table>
    <thead>
    <tr>
    <th>Endpoint</th>
    <th>Description</th>
    </tr>
    </thead>
    <tbody>
    <tr>
    <td>:9090/haproxy?stats</td>
    <td>HAProxy stats endpoint. This produces an HTML page which can be viewed in your browser, providing various statistics about the current HAProxy instance.</td>
    </tr>
    <tr>
    <td>:9090/haproxy?stats;csv</td>
    <td>This is a CSV version of the stats above, which can be consumed by other tools. For example, it's used in the zdd.py script.</td>
    </tr>
    <tr>
    <td>:9090/_haproxy_health_check</td>
    <td>HAProxy health check endpoint. Returns 200 OK if HAProxy is healthy.</td>
    </tr>
    <tr>
    <td>:9090/_haproxy_getconfig</td>
    <td>Returns the HAProxy config file as it was when HAProxy was started. Implemented in getconfig.lua.</td>
    </tr>
    <tr>
    <td>:9090/_haproxy_getvhostmap</td>
    <td>Returns the HAProxy vhost to backend map. This endpoint returns HAProxy map file only when --haproxy-map flag is enabled, it returns a empty string otherwise. Implemented in getvhostmap.lua.</td>
    </tr>
    <tr>
    <td>:9090/_haproxy_getpids</td>
    <td>Returns the PIDs for all HAProxy instances within the current process namespace. This literally returns $(pidof haproxy). Implemented in getpids.lua. This is also used by the zdd.py script to determine if connections have finished draining during a deploy.</td>
    </tr>
    </tbody>
    </table>

## 二. Nginx使用marathon-lb做负载均衡

### nginx.json

~~~
   {
     "id":"nginx",
     "labels": {
        "HAPROXY_GROUP":"external",
        "HAPROXY_0_VHOST":"nginx.marathon_ccc.mesos"
     },
     "cpus":1.0,
     "mem":128.0,
     "instances": 2,
     "healthChecks": [{ "path": "/" }],
     "container": {
       "type":"DOCKER",
       "docker": {
        "image": "nginx",
        "network": "BRIDGE",
        "portMappings":[{"containerPort":80,"hostPort":0,"servicePort":80,"protocol":"tcp"}]
       }
     }
   }
~~~


    HAPROXY_GROUP要与相应的marathon-lb的--group参数一致。

    servicePort是用于服务发现的端口配置（与docker无关），marathon不使用这个参数，marathon-lb使用。这是一个可选参数，默认为0。如果设置为0，marathon会指定一个随机的数值。

## 二. 运行机制

### 容器中进程列表

~~~
UID        PID  PPID  C STIME TTY          TIME CMD
root         1     0  0 07:30 ?        00:00:00 /bin/sh -c /marathon-lb/run sse --marathon http://192.168.36.111:8080 --group external
root         5     1  0 07:30 ?        00:00:00 /bin/bash /marathon-lb/run sse --marathon http://192.168.36.111:8080 --group external
root        12     5  0 07:30 ?        00:00:00 /usr/bin/runsv /marathon-lb/service/haproxy
root        13     5  0 07:30 ?        00:00:00 python3 /marathon-lb/marathon_lb.py --syslog-socket /dev/null --haproxy-config /marathon-lb/haproxy.cfg --ssl-certs /etc/ssl/cert.pem --command
root        14    12  0 07:30 ?        00:00:00 /bin/bash ./run
root        45     1  0 07:30 ?        00:00:00 haproxy -p /tmp/haproxy.pid -f /marathon-lb/haproxy.cfg -D -sf
root      1453    14  0 07:41 ?        00:00:00 sleep 0.5
~~~

### marathon-lb.py

~~~
    通过marathon的/v2/events接口，获取marathon中应用的部署变更事件（status_update_event、health_status_changed_event、api_post_event）。

    对marathon应用部署变更情况进行解析，更新haproxy.cfg文件。

    控制haproxy重新加载haproxy.cfg文件，使配置生效。
~~~
