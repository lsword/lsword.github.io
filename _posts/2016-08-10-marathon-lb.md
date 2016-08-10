* * *

layout: post
category: marathon-lb
tags: mesos,marathon,paas

## description: 本文介绍marathon-lb的使用

## 一. 安装部署

### 1. 获取镜像

    <span class="hljs-title">docker</span> pull mesosphere/marathon-lb
    `</pre>

    ### 2. 使用docker命令启动

    <pre>`<span class="hljs-comment">docker</span> <span class="hljs-comment">run</span> <span class="hljs-literal">-</span><span class="hljs-comment">d</span> <span class="hljs-literal">-</span><span class="hljs-comment">e</span> <span class="hljs-comment">PORTS=9090</span> <span class="hljs-literal">-</span><span class="hljs-literal">-</span><span class="hljs-comment">privileged</span> <span class="hljs-literal">-</span><span class="hljs-literal">-</span><span class="hljs-comment">net=host</span> <span class="hljs-comment">docker</span><span class="hljs-string">.</span><span class="hljs-comment">io/mesosphere/marathon</span><span class="hljs-literal">-</span><span class="hljs-comment">lb</span> <span class="hljs-comment">sse</span> <span class="hljs-literal">-</span><span class="hljs-literal">-</span><span class="hljs-comment">marathon</span> <span class="hljs-comment">http://192</span><span class="hljs-string">.</span><span class="hljs-comment">168</span><span class="hljs-string">.</span><span class="hljs-comment">36</span><span class="hljs-string">.</span><span class="hljs-comment">111:8080</span> <span class="hljs-literal">-</span><span class="hljs-literal">-</span><span class="hljs-comment">group</span> <span class="hljs-comment">external</span>
    `</pre>

    ### 3. 使用marathon启动

*   marathon-lb.json
    <pre>`{
      "<span class="hljs-attribute">id</span>": <span class="hljs-value"><span class="hljs-string">"marathon-lb"</span></span>,
      "<span class="hljs-attribute">cmd</span>": <span class="hljs-value"><span class="hljs-string">"/marathon-lb/run sse --marathon http://192.168.36.111:8080 --group external"</span></span>,
      "<span class="hljs-attribute">cpus</span>": <span class="hljs-value"><span class="hljs-number">1</span></span>,
      "<span class="hljs-attribute">mem</span>": <span class="hljs-value"><span class="hljs-number">128</span></span>,
      "<span class="hljs-attribute">disk</span>": <span class="hljs-value"><span class="hljs-number">0</span></span>,
      "<span class="hljs-attribute">instances</span>": <span class="hljs-value"><span class="hljs-number">1</span></span>,
      "<span class="hljs-attribute">container</span>": <span class="hljs-value">{
        "<span class="hljs-attribute">docker</span>": <span class="hljs-value">{
          "<span class="hljs-attribute">image</span>": <span class="hljs-value"><span class="hljs-string">"docker.io/mesosphere/marathon-lb"</span></span>,
          "<span class="hljs-attribute">network</span>": <span class="hljs-value"><span class="hljs-string">"HOST"</span></span>,
          "<span class="hljs-attribute">privileged</span>": <span class="hljs-value"><span class="hljs-literal">true</span></span>,
          "<span class="hljs-attribute">parameters</span>": <span class="hljs-value">[]
        </span>}</span>,
        "<span class="hljs-attribute">type</span>": <span class="hljs-value"><span class="hljs-string">"DOCKER"</span></span>,
        "<span class="hljs-attribute">volumes</span>": <span class="hljs-value">[]
      </span>}</span>,
      "<span class="hljs-attribute">constraints</span>": <span class="hljs-value">[
        [
          <span class="hljs-string">"hostname"</span>,
          <span class="hljs-string">"CLUSTER"</span>,
          <span class="hljs-string">"192.168.36.107"</span>
        ]
      ]</span>,
      "<span class="hljs-attribute">portDefinitions</span>": <span class="hljs-value">[
        {
          "<span class="hljs-attribute">port</span>": <span class="hljs-value"><span class="hljs-number">0</span></span>,
          "<span class="hljs-attribute">protocol</span>": <span class="hljs-value"><span class="hljs-string">"tcp"</span></span>,
          "<span class="hljs-attribute">name</span>": <span class="hljs-value"><span class="hljs-literal">null</span></span>,
          "<span class="hljs-attribute">labels</span>": <span class="hljs-value"><span class="hljs-literal">null</span>
        </span>}
      ]</span>,
      "<span class="hljs-attribute">env</span>": <span class="hljs-value">{}</span>,
      "<span class="hljs-attribute">labels</span>": <span class="hljs-value">{}
    </span>}
    `</pre>

*   curl命令
    <pre>`<span class="hljs-attribute">curl</span> -i -H <span class="hljs-string">'Content-Type: application/json'</span> <span class="hljs-number">127.0</span>.<span class="hljs-number">0.1</span>:<span class="hljs-number">8080</span>/v2/apps -d<span class="hljs-variable">@marathon-lb</span>.json
    `</pre>

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

    <pre>`{
      "<span class="hljs-attribute">id</span>":<span class="hljs-value"><span class="hljs-string">"nginx"</span></span>,
      "<span class="hljs-attribute">labels</span>": <span class="hljs-value">{
         "<span class="hljs-attribute">HAPROXY_GROUP</span>":<span class="hljs-value"><span class="hljs-string">"external"</span></span>,
         "<span class="hljs-attribute">HAPROXY_0_VHOST</span>":<span class="hljs-value"><span class="hljs-string">"nginx.marathon_ccc.mesos"</span>
      </span>}</span>,
      "<span class="hljs-attribute">cpus</span>":<span class="hljs-value"><span class="hljs-number">1.0</span></span>,
      "<span class="hljs-attribute">mem</span>":<span class="hljs-value"><span class="hljs-number">128.0</span></span>,
      "<span class="hljs-attribute">instances</span>": <span class="hljs-value"><span class="hljs-number">2</span></span>,
      "<span class="hljs-attribute">healthChecks</span>": <span class="hljs-value">[{ "<span class="hljs-attribute">path</span>": <span class="hljs-value"><span class="hljs-string">"/"</span> </span>}]</span>,
      "<span class="hljs-attribute">container</span>": <span class="hljs-value">{
        "<span class="hljs-attribute">type</span>":<span class="hljs-value"><span class="hljs-string">"DOCKER"</span></span>,
        "<span class="hljs-attribute">docker</span>": <span class="hljs-value">{
         "<span class="hljs-attribute">image</span>": <span class="hljs-value"><span class="hljs-string">"nginx"</span></span>,
         "<span class="hljs-attribute">network</span>": <span class="hljs-value"><span class="hljs-string">"BRIDGE"</span></span>,
         "<span class="hljs-attribute">portMappings</span>":<span class="hljs-value">[{"<span class="hljs-attribute">containerPort</span>":<span class="hljs-value"><span class="hljs-number">80</span></span>,"<span class="hljs-attribute">hostPort</span>":<span class="hljs-value"><span class="hljs-number">0</span></span>,"<span class="hljs-attribute">servicePort</span>":<span class="hljs-value"><span class="hljs-number">80</span></span>,"<span class="hljs-attribute">protocol</span>":<span class="hljs-value"><span class="hljs-string">"tcp"</span></span>}]
        </span>}
      </span>}
    </span>}
    `</pre>

    HAPROXY_GROUP要与相应的marathon-lb的--group参数一致。

    servicePort是用于服务发现的端口配置（与docker无关），marathon不使用这个参数，marathon-lb使用。这是一个可选参数，默认为0。如果设置为0，marathon会指定一个随机的数值。

    ## 二. 运行机制

    ### 容器中进程列表

    <pre>`UID        PID  PPID  C STIME TTY          TIME CMD
    root         <span class="hljs-number">1</span>     <span class="hljs-number">0</span>  <span class="hljs-number">0</span> <span class="hljs-number">07</span>:<span class="hljs-number">30</span> ?        00:<span class="hljs-number">00</span>:<span class="hljs-number">00</span> <span class="hljs-regexp">/bin/</span>sh -c <span class="hljs-regexp">/marathon-lb/</span>run sse --marathon <span class="hljs-string">http:</span><span class="hljs-comment">//192.168.36.111:8080 --group external</span>
    root         <span class="hljs-number">5</span>     <span class="hljs-number">1</span>  <span class="hljs-number">0</span> <span class="hljs-number">07</span>:<span class="hljs-number">30</span> ?        00:<span class="hljs-number">00</span>:<span class="hljs-number">00</span> <span class="hljs-regexp">/bin/</span>bash <span class="hljs-regexp">/marathon-lb/</span>run sse --marathon <span class="hljs-string">http:</span><span class="hljs-comment">//192.168.36.111:8080 --group external</span>
    root        <span class="hljs-number">12</span>     <span class="hljs-number">5</span>  <span class="hljs-number">0</span> <span class="hljs-number">07</span>:<span class="hljs-number">30</span> ?        00:<span class="hljs-number">00</span>:<span class="hljs-number">00</span> <span class="hljs-regexp">/usr/</span>bin<span class="hljs-regexp">/runsv /</span>marathon-lb<span class="hljs-regexp">/service/</span>haproxy
    root        <span class="hljs-number">13</span>     <span class="hljs-number">5</span>  <span class="hljs-number">0</span> <span class="hljs-number">07</span>:<span class="hljs-number">30</span> ?        00:<span class="hljs-number">00</span>:<span class="hljs-number">00</span> python3 <span class="hljs-regexp">/marathon-lb/</span>marathon_lb.py --syslog-socket <span class="hljs-regexp">/dev/</span><span class="hljs-literal">null</span> --haproxy-config <span class="hljs-regexp">/marathon-lb/</span>haproxy.cfg --ssl-certs <span class="hljs-regexp">/etc/</span>ssl/cert.pem --command
    root        <span class="hljs-number">14</span>    <span class="hljs-number">12</span>  <span class="hljs-number">0</span> <span class="hljs-number">07</span>:<span class="hljs-number">30</span> ?        00:<span class="hljs-number">00</span>:<span class="hljs-number">00</span> <span class="hljs-regexp">/bin/</span>bash ./run
    root        <span class="hljs-number">45</span>     <span class="hljs-number">1</span>  <span class="hljs-number">0</span> <span class="hljs-number">07</span>:<span class="hljs-number">30</span> ?        00:<span class="hljs-number">00</span>:<span class="hljs-number">00</span> haproxy -p <span class="hljs-regexp">/tmp/</span>haproxy.pid -f <span class="hljs-regexp">/marathon-lb/</span>haproxy.cfg -D -sf
    root      <span class="hljs-number">1453</span>    <span class="hljs-number">14</span>  <span class="hljs-number">0</span> <span class="hljs-number">07</span>:<span class="hljs-number">41</span> ?        00:<span class="hljs-number">00</span>:<span class="hljs-number">00</span> sleep <span class="hljs-number">0.5</span>
    `</pre>

    ### marathon-lb.py

    <pre>`通过marathon的<span class="hljs-regexp">/v2/</span>events接口，获取marathon中应用的部署变更事件（status_update_event、health_status_changed_event、api_post_event）。

    对marathon应用部署变更情况进行解析，更新haproxy.cfg文件。

    控制haproxy重新加载haproxy.cfg文件，使配置生效。
    