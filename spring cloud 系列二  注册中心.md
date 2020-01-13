# Spring Cloud 微服务系列二：注册中心



> Spring Cloud Consul：封装了Consul操作，consul是一个服务发现与配置工具，与Docker容器可以无缝集成。
>
> Consul 简化了分布式环境中的服务的注册和发现流程，通过 HTTP 或者 DNS 接口发现。支持外部 SaaS 提供者等。
>
> consul提供的一些关键特性：
>
> service discovery：consul通过DNS或者HTTP接口使服务注册和服务发现变的很容易，一些外部服务，例如saas提供的也可以一样注册。
>
> health checking：健康检测使consul可以快速的告警在集群中的操作。和服务发现的集成，可以防止服务转发到故障的服务上面。
>
> key/value storage：一个用来存储动态配置的系统。提供简单的HTTP接口，可以在任何地方操作。
>
> multi-datacenter：无需复杂的配置，即可支持任意数量的区域(数据中心)。
>
> 官方网站：https://www.consul.io/



## 术语

- 代理 - 代理是指consul集群中运行的consul实例，通过执行 `consul agent 命令来启动`. 代理可以运行于客户端或者服务端。通过执行DNS或者HTTP接口来执行健康检查和服务同步。

- 客户节点 - 客户节点负责将向服务节点发送RPC请求，相对来说是无状态的.。客户节点唯一要执行的后台活动是参与LAN gossip pool。当然这只会消耗很少的资源和网络带宽。

- 服务节点 - 服务节点主要职责包括参与Raft算法，维护集群状态，处理RPC查询，和其它的数据中心交换WLAN gossip及向领导者或者远程数据中心转发查询请求。

- 数据中心 – …

- 维护一致性- 包括领导者选举及事务执行顺序方面的一致性。

- Gossip - Consul是建立在Serf之上的，支持全部的[gossip protocol](https://en.wikipedia.org/wiki/Gossip_protocol)（节点间随机通信，主要通过UDP）。Serf 提供关系管理, 失败检测及事件分发功能. Consul gossip应用详情可以访问连接[gossip documentation](https://www.consul.io/docs/internals/gossip.html)。

- LAN Gossip - 本地局域网或数据中心节点间Gossip应用.

- WAN Gossip – WAN 范围（不同数据中心，主要通过internet或者wan通信）内服务器节点间Gossip应用。

- RPC - 远程过程调用，请求/回复机制。

  



## 下载地址

>上篇文章有介绍版本 spring boot & cloud 关系, 本次使用 consul 对应版本 1.2.x
>
>下载网址： https://releases.hashicorp.com/consul/1.2.0/ 



## Consul 文件

>  下载解压，里面只有一个consul可执行文件

### 1. windows 版本

> 解压缩 Windows + R    输入： cmd

```cmd
D:\Consul>consul
usage: consul [--version] [--help] <command> [<args>]

Available commands are:
    agent          Runs a Consul agent
    configtest     Validate config file
    event          Fire a new event
    exec           Executes a command on Consul nodes
    force-leave    Forces a member of the cluster to enter the "left" state
    info           Provides debugging information for operators
    join           Tell Consul agent to join cluster
    keygen         Generates a new encryption key
    keyring        Manages gossip layer encryption keys
    kv             Interact with the key-value store
    leave          Gracefully leaves the Consul cluster and shuts down
    lock           Execute a command holding a lock
    maint          Controls node or service maintenance mode
    members        Lists the members of a Consul cluster
    monitor        Stream logs from a Consul agent
    operator       Provides cluster-level tools for Consul operators
    reload         Triggers the agent to reload configuration files
    rtt            Estimates network round trip time between nodes
    snapshot       Saves, restores and inspects snapshots of Consul server state
    version        Prints the Consul version
    watch          Watch for changes in Consul


D:\Consul>
```







### 2. Linux版本

```shell
[root@master zhangwei]# unzip consul_1.2.0_linux_amd64.zip 
Archive:  consul_1.2.0_linux_amd64.zip
  inflating: consul                  
[root@master zhangwei]# 
```





```shell
[root@master zhangwei]# ./consul 
Usage: consul [--version] [--help] <command> [<args>]

Available commands are:
    agent          Runs a Consul agent
    catalog        Interact with the catalog
    connect        Interact with Consul Connect
    event          Fire a new event
    exec           Executes a command on Consul nodes
    force-leave    Forces a member of the cluster to enter the "left" state
    info           Provides debugging information for operators.
    intention      Interact with Connect service intentions
    join           Tell Consul agent to join cluster
    keygen         Generates a new encryption key
    keyring        Manages gossip layer encryption keys
    kv             Interact with the key-value store
    leave          Gracefully leaves the Consul cluster and shuts down
    lock           Execute a command holding a lock
    maint          Controls node or service maintenance mode
    members        Lists the members of a Consul cluster
    monitor        Stream logs from a Consul agent
    operator       Provides cluster-level tools for Consul operators
    reload         Triggers the agent to reload configuration files
    rtt            Estimates network round trip time between nodes
    snapshot       Saves, restores and inspects snapshots of Consul server state
    validate       Validate config files/directories
    version        Prints the Consul version
    watch          Watch for changes in Consul

[root@master zhangwei]#
```





### 3. 总结： 

> 无论什么版本，下载后输入以上演示命令表示环境成功【如果是linux 环境建议添加环境变量】





## Consul 命令

### 1.   常见的参数解释

```txt
-advertise：通知展现地址用来改变我们给集群中的其他节点展现的地址，一般情况下-bind地址就是展现地址
-bootstrap：用来控制一个server是否在bootstrap模式，在一个datacenter中只能有一个server处于bootstrap模式，当一个server处于bootstrap模式时，可以自己选举为raft leader。
-bootstrap-expect：在一个datacenter中期望提供的server节点数目，当该值提供的时候，consul一直等到达到指定sever数目的时候才会引导整个集群，该标记不能和bootstrap公用
-bind：该地址用来在集群内部的通讯，集群内的所有节点到地址都必须是可达的，默认是0.0.0.0
-client：consul绑定在哪个client地址上，这个地址提供HTTP、DNS、RPC等服务，默认是127.0.0.1
-config-file：明确的指定要加载哪个配置文件
-config-dir：配置文件目录，里面所有以.json结尾的文件都会被加载
-data-dir：提供一个目录用来存放agent的状态，所有的agent允许都需要该目录，该目录必须是稳定的，系统重启后都继续存在
-dc：该标记控制agent允许的datacenter的名称，默认是dc1
-encrypt：指定secret key，使consul在通讯时进行加密，key可以通过consul keygen生成，同一个集群中的节点必须使用相同的key
-join：加入一个已经启动的agent的ip地址，可以多次指定多个agent的地址。如果consul不能加入任何指定的地址中，则agent会启动失败，默认agent启动时不会加入任何节点。
-retry-join：和join类似，但是允许你在第一次失败后进行尝试。
-retry-interval：两次join之间的时间间隔，默认是30s
-retry-max：尝试重复join的次数，默认是0，也就是无限次尝试
-log-level：consul agent启动后显示的日志信息级别。默认是info，可选：trace、debug、info、warn、err。
-node：节点在集群中的名称，在一个集群中必须是唯一的，默认是该节点的主机名
-protocol：consul使用的协议版本
-rejoin：使consul忽略先前的离开，在再次启动后仍旧尝试加入集群中。
-server：定义agent运行在server模式，每个集群至少有一个server，建议每个集群的server不要超过5个
-syslog：开启系统日志功能，只在linux/osx上生效
-ui-dir: 提供存放web ui资源的路径，该目录必须是可读的
-pid-file: 提供一个路径来存放pid文件，可以使用该文件进行SIGINT/SIGHUP(关闭/更新)agent
```



### 2. 启动consul 开发模式

```shell
[root@iZuf655czz7lmnhrn9p1cuZ zhangwei]# ./consul agent -dev -client=0.0.0.0 -ui
==> Starting Consul agent...
==> Consul agent running!
           Version: 'v1.4.0'
           Node ID: 'e94ffe06-b59c-6158-3f05-0eb23564951b'
         Node name: 'iZuf655czz7lmnhrn9p1cuZ'
        Datacenter: 'dc1' (Segment: '<all>')
            Server: true (Bootstrap: false)
       Client Addr: [0.0.0.0] (HTTP: 8500, HTTPS: -1, gRPC: 8502, DNS: 8600)
      Cluster Addr: 127.0.0.1 (LAN: 8301, WAN: 8302)
           Encrypt: Gossip: false, TLS-Outgoing: false, TLS-Incoming: false

==> Log data will now stream in as it occurs:

    2020/01/12 16:10:18 [DEBUG] agent: Using random ID "e94ffe06-b59c-6158-3f05-0eb23564951b" as node ID
    2020/01/12 16:10:18 [INFO] raft: Initial configuration (index=1): [{Suffrage:Voter ID:e94ffe06-b59c-6158-3f05-0eb23564951b Address:127.0.0.1:8300}]
    2020/01/12 16:10:18 [INFO] serf: EventMemberJoin: iZuf655czz7lmnhrn9p1cuZ.dc1 127.0.0.1
    2020/01/12 16:10:18 [INFO] serf: EventMemberJoin: iZuf655czz7lmnhrn9p1cuZ 127.0.0.1
    2020/01/12 16:10:18 [INFO] agent: Started DNS server 0.0.0.0:8600 (udp)
    2020/01/12 16:10:18 [INFO] raft: Node at 127.0.0.1:8300 [Follower] entering Follower state (Leader: "")
    2020/01/12 16:10:18 [INFO] consul: Adding LAN server iZuf655czz7lmnhrn9p1cuZ (Addr: tcp/127.0.0.1:8300) (DC: dc1)
    2020/01/12 16:10:18 [INFO] consul: Handled member-join event for server "iZuf655czz7lmnhrn9p1cuZ.dc1" in area "wan"
    2020/01/12 16:10:18 [DEBUG] agent/proxy: managed Connect proxy manager started
    2020/01/12 16:10:18 [WARN] agent/proxy: running as root, will not start managed proxies
    2020/01/12 16:10:18 [INFO] agent: Started DNS server 0.0.0.0:8600 (tcp)
    2020/01/12 16:10:18 [INFO] agent: Started HTTP server on [::]:8500 (tcp)
    2020/01/12 16:10:18 [INFO] agent: started state syncer
    2020/01/12 16:10:18 [INFO] agent: Started gRPC server on [::]:8502 (tcp)
    2020/01/12 16:10:18 [WARN] raft: Heartbeat timeout from "" reached, starting election
    2020/01/12 16:10:18 [INFO] raft: Node at 127.0.0.1:8300 [Candidate] entering Candidate state in term 2
    2020/01/12 16:10:18 [DEBUG] raft: Votes needed: 1
    2020/01/12 16:10:18 [DEBUG] raft: Vote granted from e94ffe06-b59c-6158-3f05-0eb23564951b in term 2. Tally: 1
    2020/01/12 16:10:18 [INFO] raft: Election won. Tally: 1
    2020/01/12 16:10:18 [INFO] raft: Node at 127.0.0.1:8300 [Leader] entering Leader state
    2020/01/12 16:10:18 [INFO] consul: cluster leadership acquired
    2020/01/12 16:10:18 [INFO] connect: initialized primary datacenter CA with provider "consul"
    2020/01/12 16:10:18 [DEBUG] consul: Skipping self join check for "iZuf655czz7lmnhrn9p1cuZ" since the cluster is too small
    2020/01/12 16:10:18 [INFO] consul: member 'iZuf655czz7lmnhrn9p1cuZ' joined, marking health alive
    2020/01/12 16:10:18 [INFO] consul: New leader elected: iZuf655czz7lmnhrn9p1cuZ
    2020/01/12 16:10:19 [DEBUG] agent: Skipping remote check "serfHealth" since it is managed automatically
    2020/01/12 16:10:19 [INFO] agent: Synced node info
    2020/01/12 16:10:19 [DEBUG] agent: Node info in sync
    2020/01/12 16:10:19 [DEBUG] agent: Skipping remote check "serfHealth" since it is managed automatically
    2020/01/12 16:10:19 [DEBUG] agent: Node info in sync
```



> 此时访问: http://ip:8500  出现Consul UI界面表示成功 (windows 版本命令一样)



## 集群搭建

### 1.  **Consul的agent角色** 

  Consul的agent分成Server和Client两种角色，不论Server和Client都是Consul的节点，所有的服务都可以注册到这些节点上，通过这些节点实现服务注册信息的共享。

  Server和Client的区别在于，Server会把信息持久化到本地保存，便于故障后的恢复。而Client则是把服务信息转发到Server，本身并不做持久化。

 ![img](https://img-blog.csdn.net/2018022623032414) 



### 2.  局域网主机搭建集群

> 1.  192.168.203.133
> 2.  192.168.203.134
> 3.  192.168.203.135



### 3.  启动集群

```shell
./consul agent -server  -bootstrap-expect 3 -data-dir=/zhangwei/data -bind=192.168.203.133 -client=0.0.0.0 -ui -node=192.168.203.133

./consul agent -server  -bootstrap-expect 3 -data-dir=/zhangwei/data -advertise=192.168.203.134 -client=0.0.0.0 -node=192.168.203.134 -join=192.168.203.133

./consul agent -server  -bootstrap-expect 3 -data-dir=/zhangwei/data -advertise=192.168.203.135 -client=0.0.0.0 -node=192.168.203.135 -join=192.168.203.133
```



```shell
[root@master zhangwei]# ./consul agent -server  -bootstrap-expect 3 -data-dir=/zhangwei/data -bind=192.168.203.133 -client=0.0.0.0 -ui -node=192.168.203.133
bootstrap_expect > 0: expecting 3 servers
==> Starting Consul agent...
==> Consul agent running!
           Version: 'v1.2.0'
           Node ID: '57965f19-ecd6-eb1f-ad86-6993242862c0'
         Node name: '192.168.203.133'
        Datacenter: 'dc1' (Segment: '<all>')
            Server: true (Bootstrap: false)
       Client Addr: [0.0.0.0] (HTTP: 8500, HTTPS: -1, DNS: 8600)
      Cluster Addr: 192.168.203.133 (LAN: 8301, WAN: 8302)
           Encrypt: Gossip: false, TLS-Outgoing: false, TLS-Incoming: false

==> Log data will now stream in as it occurs:

    2020/01/12 18:15:01 [WARN] agent: Node name "192.168.203.133" will not be discoverable via DNS due to invalid characters. Valid characters include all alpha-numerics and dashes.
    2020/01/12 18:15:01 [INFO] raft: Initial configuration (index=1): [{Suffrage:Voter ID:57965f19-ecd6-eb1f-ad86-6993242862c0 Address:192.168.203.133:8300} {Suffrage:Voter ID:580d0fdf-dc4b-6d2a-ae97-4e4664cbb49f Address:192.168.203.134:8300} {Suffrage:Voter ID:a8b1440a-1ce9-9e72-9e08-717f9170e4eb Address:192.168.203.135:8300}]
    2020/01/12 18:15:01 [INFO] raft: Node at 192.168.203.133:8300 [Follower] entering Follower state (Leader: "")
    2020/01/12 18:15:01 [INFO] serf: EventMemberJoin: 192.168.203.133.dc1 192.168.203.133
    2020/01/12 18:15:01 [INFO] serf: Attempting re-join to previously known node: 192.168.203.135.dc1: 192.168.203.135:8302
    2020/01/12 18:15:01 [INFO] serf: EventMemberJoin: 192.168.203.133 192.168.203.133
    2020/01/12 18:15:01 [INFO] serf: EventMemberJoin: 192.168.203.135.dc1 192.168.203.135
    2020/01/12 18:15:01 [WARN] memberlist: Refuting a suspect message (from: 192.168.203.133.dc1)
    2020/01/12 18:15:01 [INFO] serf: EventMemberJoin: 192.168.203.134.dc1 192.168.203.134
    2020/01/12 18:15:01 [INFO] serf: Re-joined to previously known node: 192.168.203.135.dc1: 192.168.203.135:8302
    2020/01/12 18:15:01 [INFO] consul: Handled member-join event for server "192.168.203.133.dc1" in area "wan"
    2020/01/12 18:15:01 [INFO] consul: Handled member-join event for server "192.168.203.135.dc1" in area "wan"
    2020/01/12 18:15:01 [INFO] consul: Handled member-join event for server "192.168.203.134.dc1" in area "wan"
    2020/01/12 18:15:01 [INFO] serf: Attempting re-join to previously known node: 192.168.203.135: 192.168.203.135:8301
    2020/01/12 18:15:01 [INFO] consul: Adding LAN server 192.168.203.133 (Addr: tcp/192.168.203.133:8300) (DC: dc1)
    2020/01/12 18:15:01 [INFO] agent: Started DNS server 0.0.0.0:8600 (udp)
    2020/01/12 18:15:01 [INFO] consul: Raft data found, disabling bootstrap mode
    2020/01/12 18:15:01 [WARN] agent/proxy: running as root, will not start managed proxies
    2020/01/12 18:15:01 [INFO] agent: Started DNS server 0.0.0.0:8600 (tcp)
    2020/01/12 18:15:01 [INFO] agent: Started HTTP server on [::]:8500 (tcp)
    2020/01/12 18:15:01 [INFO] agent: started state syncer
    2020/01/12 18:15:01 [INFO] serf: EventMemberJoin: 192.168.203.134 192.168.203.134
    2020/01/12 18:15:01 [INFO] serf: EventMemberJoin: 192.168.203.135 192.168.203.135
    2020/01/12 18:15:01 [INFO] serf: Re-joined to previously known node: 192.168.203.135: 192.168.203.135:8301
    2020/01/12 18:15:01 [INFO] consul: Adding LAN server 192.168.203.134 (Addr: tcp/192.168.203.134:8300) (DC: dc1)
    2020/01/12 18:15:01 [INFO] consul: Adding LAN server 192.168.203.135 (Addr: tcp/192.168.203.135:8300) (DC: dc1)
    2020/01/12 18:15:01 [INFO] consul: New leader elected: 192.168.203.135
    2020/01/12 18:15:08 [WARN] raft: Heartbeat timeout from "" reached, starting election
    2020/01/12 18:15:08 [INFO] raft: Node at 192.168.203.133:8300 [Candidate] entering Candidate state in term 3
    2020/01/12 18:15:08 [INFO] raft: Node at 192.168.203.133:8300 [Follower] entering Follower state (Leader: "")
    2020/01/12 18:15:08 [ERR] agent: failed to sync remote state: No cluster leader
2020/01/12 18:15:10 [DEBUG] raft-net: 192.168.203.133:8300 accepted connection from: 192.168.203.135:50061
    2020/01/12 18:15:10 [INFO] agent: Synced node info
2020/01/12 18:15:10 [DEBUG] raft-net: 192.168.203.133:8300 accepted connection from: 192.168.203.135:42034
==> Newer Consul version available: 1.6.2 (currently running: 1.2.0)

```





```shell
[root@master zhangwei]# ./consul agent -server  -bootstrap-expect 3 -data-dir=/zhangwei/data -bind=192.168.203.134 -client=0.0.0.0 -node=192.168.203.134 -join=192.168.203.133
bootstrap_expect > 0: expecting 3 servers
==> Starting Consul agent...
==> Joining cluster...
    Join completed. Synced with 1 initial agents
==> Consul agent running!
           Version: 'v1.2.0'
           Node ID: '580d0fdf-dc4b-6d2a-ae97-4e4664cbb49f'
         Node name: '192.168.203.134'
        Datacenter: 'dc1' (Segment: '<all>')
            Server: true (Bootstrap: false)
       Client Addr: [0.0.0.0] (HTTP: 8500, HTTPS: -1, DNS: 8600)
      Cluster Addr: 192.168.203.134 (LAN: 8301, WAN: 8302)
           Encrypt: Gossip: false, TLS-Outgoing: false, TLS-Incoming: false

==> Log data will now stream in as it occurs:

    2020/01/12 18:09:40 [WARN] agent: Node name "192.168.203.134" will not be discoverable via DNS due to invalid characters. Valid characters include all alpha-numerics and dashes.
    2020/01/12 18:09:40 [INFO] raft: Initial configuration (index=0): []
    2020/01/12 18:09:40 [INFO] raft: Node at 192.168.203.134:8300 [Follower] entering Follower state (Leader: "")
    2020/01/12 18:09:40 [INFO] serf: EventMemberJoin: 192.168.203.134.dc1 192.168.203.134
    2020/01/12 18:09:40 [WARN] serf: Failed to re-join any previously known node
    2020/01/12 18:09:40 [INFO] serf: EventMemberJoin: 192.168.203.134 192.168.203.134
    2020/01/12 18:09:40 [INFO] consul: Adding LAN server 192.168.203.134 (Addr: tcp/192.168.203.134:8300) (DC: dc1)
    2020/01/12 18:09:40 [INFO] consul: Handled member-join event for server "192.168.203.134.dc1" in area "wan"
    2020/01/12 18:09:40 [WARN] serf: Failed to re-join any previously known node
    2020/01/12 18:09:40 [WARN] agent/proxy: running as root, will not start managed proxies
    2020/01/12 18:09:40 [INFO] agent: Started DNS server 0.0.0.0:8600 (tcp)
    2020/01/12 18:09:40 [INFO] agent: Started DNS server 0.0.0.0:8600 (udp)
    2020/01/12 18:09:40 [INFO] agent: Started HTTP server on [::]:8500 (tcp)
    2020/01/12 18:09:40 [INFO] agent: (LAN) joining: [192.168.203.133]    2020/01/12 18:09:40 [INFO] consul: Adding LAN server 192.168.203.133 (Addr: tcp/192.168.203.133:8300) (DC: dc1)

    2020/01/12 18:09:40 [INFO] serf: EventMemberJoin: 192.168.203.133 192.168.203.133
    2020/01/12 18:09:40 [INFO] agent: (LAN) joined: 1 Err: <nil>
    2020/01/12 18:09:40 [INFO] agent: started state syncer
    2020/01/12 18:09:40 [INFO] serf: EventMemberJoin: 192.168.203.133.dc1 192.168.203.133
    2020/01/12 18:09:40 [INFO] consul: Handled member-join event for server "192.168.203.133.dc1" in area "wan"
==> Failed to check for updates: Get https://checkpoint-api.hashicorp.com/v1/check/consul?arch=amd64&os=linux&signature=882d0ce1-bde9-9d05-2606-90c9e664aed4&version=1.2.0: net/http: request canceled while waiting for connection (Client.Timeout exceeded while awaiting headers)
    2020/01/12 18:09:47 [ERR] agent: failed to sync remote state: No cluster leader
    2020/01/12 18:09:49 [WARN] raft: no known peers, aborting election
    2020/01/12 18:09:53 [INFO] serf: EventMemberJoin: 192.168.203.135.dc1 192.168.203.135
    2020/01/12 18:09:53 [INFO] consul: Handled member-join event for server "192.168.203.135.dc1" in area "wan"
    2020/01/12 18:09:53 [INFO] serf: EventMemberJoin: 192.168.203.135 192.168.203.135
    2020/01/12 18:09:53 [INFO] consul: Adding LAN server 192.168.203.135 (Addr: tcp/192.168.203.135:8300) (DC: dc1)
    2020/01/12 18:09:53 [INFO] consul: Existing Raft peers reported by 192.168.203.133, disabling bootstrap mode
2020/01/12 18:09:54 [DEBUG] raft-net: 192.168.203.134:8300 accepted connection from: 192.168.203.133:51826
2020/01/12 18:09:54 [DEBUG] raft-net: 192.168.203.134:8300 accepted connection from: 192.168.203.133:54887
    2020/01/12 18:09:54 [WARN] raft: Failed to get previous log: 1 log not found (last: 0)
    2020/01/12 18:09:54 [INFO] consul: New leader elected: 192.168.203.133
    2020/01/12 18:09:55 [INFO] agent: Synced node info
    2020/01/12 18:14:05 [ERR] agent: Coordinate update error: rpc error making call: stream closed
    2020/01/12 18:14:07 [INFO] memberlist: Suspect 192.168.203.133 has failed, no acks received
    2020/01/12 18:14:09 [INFO] memberlist: Suspect 192.168.203.133 has failed, no acks received
    2020/01/12 18:14:10 [INFO] memberlist: Suspect 192.168.203.133.dc1 has failed, no acks received
2020/01/12 18:14:10 [DEBUG] raft-net: 192.168.203.134:8300 accepted connection from: 192.168.203.135:48380
    2020/01/12 18:14:10 [WARN] raft: Rejecting vote request from 192.168.203.135:8300 since we have a leader: 192.168.203.133:8300
    2020/01/12 18:14:10 [INFO] memberlist: Marking 192.168.203.133 as failed, suspect timeout reached (0 peer confirmations)
    2020/01/12 18:14:10 [INFO] serf: EventMemberFailed: 192.168.203.133 192.168.203.133
    2020/01/12 18:14:10 [INFO] consul: Removing LAN server 192.168.203.133 (Addr: tcp/192.168.203.133:8300) (DC: dc1)
    2020/01/12 18:14:11 [INFO] memberlist: Suspect 192.168.203.133 has failed, no acks received
    2020/01/12 18:14:16 [WARN] raft: Heartbeat timeout from "192.168.203.133:8300" reached, starting election
    2020/01/12 18:14:16 [INFO] raft: Node at 192.168.203.134:8300 [Candidate] entering Candidate state in term 3
2020/01/12 18:14:16 [WARN] Unable to get address for server id 57965f19-ecd6-eb1f-ad86-6993242862c0, using fallback address 192.168.203.133:8300: Could not find address for server id 57965f19-ecd6-eb1f-ad86-6993242862c0
    2020/01/12 18:14:16 [ERR] raft: Failed to make RequestVote RPC to {Voter 57965f19-ecd6-eb1f-ad86-6993242862c0 192.168.203.133:8300}: dial tcp 192.168.203.134:0->192.168.203.133:8300: connect: connection refused
    2020/01/12 18:14:18 [INFO] raft: Node at 192.168.203.134:8300 [Follower] entering Follower state (Leader: "")
    2020/01/12 18:14:18 [INFO] consul: New leader elected: 192.168.203.135
2020/01/12 18:14:19 [DEBUG] raft-net: 192.168.203.134:8300 accepted connection from: 192.168.203.135:38432
    2020/01/12 18:14:20 [INFO] memberlist: Suspect 192.168.203.133.dc1 has failed, no acks received
    2020/01/12 18:14:35 [INFO] memberlist: Suspect 192.168.203.133.dc1 has failed, no acks received
    2020/01/12 18:14:40 [INFO] memberlist: Marking 192.168.203.133.dc1 as failed, suspect timeout reached (0 peer confirmations)
    2020/01/12 18:14:40 [INFO] serf: EventMemberFailed: 192.168.203.133.dc1 192.168.203.133
    2020/01/12 18:14:40 [INFO] consul: Handled member-failed event for server "192.168.203.133.dc1" in area "wan"
    2020/01/12 18:14:40 [INFO] serf: attempting reconnect to 192.168.203.133.dc1 192.168.203.133:8302
    2020/01/12 18:14:45 [INFO] memberlist: Suspect 192.168.203.133.dc1 has failed, no acks received
    2020/01/12 18:15:01 [INFO] serf: EventMemberJoin: 192.168.203.133 192.168.203.133
    2020/01/12 18:15:01 [INFO] consul: Adding LAN server 192.168.203.133 (Addr: tcp/192.168.203.133:8300) (DC: dc1)
    2020/01/12 18:15:02 [INFO] serf: EventMemberJoin: 192.168.203.133.dc1 192.168.203.133
    2020/01/12 18:15:02 [INFO] consul: Handled member-join event for server "192.168.203.133.dc1" in area "wan"
2020/01/12 18:15:08 [DEBUG] raft-net: 192.168.203.134:8300 accepted connection from: 192.168.203.133:51164
    2020/01/12 18:15:08 [WARN] raft: Rejecting vote request from 192.168.203.133:8300 since we have a leader: 192.168.203.135:8300
```



```shell
[root@master zhangwei]# ./consul agent -server  -bootstrap-expect 3 -data-dir=/zhangwei/data -bind=192.168.203.135 -client=0.0.0.0 -node=192.168.203.135 -join=192.168.203.133
bootstrap_expect > 0: expecting 3 servers
==> Starting Consul agent...
==> Joining cluster...
    Join completed. Synced with 1 initial agents
==> Consul agent running!
           Version: 'v1.2.0'
           Node ID: 'a8b1440a-1ce9-9e72-9e08-717f9170e4eb'
         Node name: '192.168.203.135'
        Datacenter: 'dc1' (Segment: '<all>')
            Server: true (Bootstrap: false)
       Client Addr: [0.0.0.0] (HTTP: 8500, HTTPS: -1, DNS: 8600)
      Cluster Addr: 192.168.203.135 (LAN: 8301, WAN: 8302)
           Encrypt: Gossip: false, TLS-Outgoing: false, TLS-Incoming: false

==> Log data will now stream in as it occurs:

    2020/01/12 18:09:53 [WARN] agent: Node name "192.168.203.135" will not be discoverable via DNS due to invalid characters. Valid characters include all alpha-numerics and dashes.
    2020/01/12 18:09:53 [INFO] raft: Initial configuration (index=0): []
    2020/01/12 18:09:53 [INFO] raft: Node at 192.168.203.135:8300 [Follower] entering Follower state (Leader: "")
    2020/01/12 18:09:53 [INFO] serf: EventMemberJoin: 192.168.203.135.dc1 192.168.203.135
    2020/01/12 18:09:53 [WARN] serf: Failed to re-join any previously known node
    2020/01/12 18:09:53 [INFO] serf: EventMemberJoin: 192.168.203.135 192.168.203.135
    2020/01/12 18:09:53 [WARN] serf: Failed to re-join any previously known node
    2020/01/12 18:09:53 [INFO] consul: Adding LAN server 192.168.203.135 (Addr: tcp/192.168.203.135:8300) (DC: dc1)
    2020/01/12 18:09:53 [INFO] consul: Handled member-join event for server "192.168.203.135.dc1" in area "wan"
    2020/01/12 18:09:53 [INFO] agent: Started DNS server 0.0.0.0:8600 (udp)
    2020/01/12 18:09:53 [WARN] agent/proxy: running as root, will not start managed proxies
    2020/01/12 18:09:53 [INFO] agent: Started DNS server 0.0.0.0:8600 (tcp)
    2020/01/12 18:09:53 [INFO] agent: Started HTTP server on [::]:8500 (tcp)
    2020/01/12 18:09:53 [INFO] agent: (LAN) joining: [192.168.203.133]
    2020/01/12 18:09:53 [INFO] serf: EventMemberJoin: 192.168.203.133 192.168.203.133
    2020/01/12 18:09:53 [INFO] consul: Adding LAN server 192.168.203.133 (Addr: tcp/192.168.203.133:8300) (DC: dc1)
    2020/01/12 18:09:53 [INFO] serf: EventMemberJoin: 192.168.203.134 192.168.203.134
    2020/01/12 18:09:53 [INFO] agent: (LAN) joined: 1 Err: <nil>
    2020/01/12 18:09:53 [INFO] agent: started state syncer
    2020/01/12 18:09:53 [INFO] consul: Existing Raft peers reported by 192.168.203.133, disabling bootstrap mode
    2020/01/12 18:09:53 [INFO] consul: Adding LAN server 192.168.203.134 (Addr: tcp/192.168.203.134:8300) (DC: dc1)
    2020/01/12 18:09:53 [INFO] serf: EventMemberJoin: 192.168.203.133.dc1 192.168.203.133
    2020/01/12 18:09:53 [INFO] serf: EventMemberJoin: 192.168.203.134.dc1 192.168.203.134
    2020/01/12 18:09:53 [INFO] consul: Handled member-join event for server "192.168.203.133.dc1" in area "wan"
    2020/01/12 18:09:53 [INFO] consul: Handled member-join event for server "192.168.203.134.dc1" in area "wan"
2020/01/12 18:09:54 [DEBUG] raft-net: 192.168.203.135:8300 accepted connection from: 192.168.203.133:37328
    2020/01/12 18:09:54 [WARN] raft: Failed to get previous log: 1 log not found (last: 0)
    2020/01/12 18:09:54 [INFO] agent: Synced node info
    2020/01/12 18:09:54 [INFO] consul: New leader elected: 192.168.203.133
==> Newer Consul version available: 1.6.2 (currently running: 1.2.0)
2020/01/12 18:09:55 [DEBUG] raft-net: 192.168.203.135:8300 accepted connection from: 192.168.203.133:50598
    2020/01/12 18:14:06 [INFO] memberlist: Suspect 192.168.203.133 has failed, no acks received
    2020/01/12 18:14:08 [ERR] agent: Coordinate update error: rpc error making call: stream closed
    2020/01/12 18:14:09 [INFO] memberlist: Suspect 192.168.203.133 has failed, no acks received
    2020/01/12 18:14:10 [WARN] raft: Heartbeat timeout from "192.168.203.133:8300" reached, starting election
    2020/01/12 18:14:10 [INFO] raft: Node at 192.168.203.135:8300 [Candidate] entering Candidate state in term 3
    2020/01/12 18:14:10 [ERR] raft: Failed to make RequestVote RPC to {Voter 57965f19-ecd6-eb1f-ad86-6993242862c0 192.168.203.133:8300}: dial tcp 192.168.203.135:0->192.168.203.133:8300: connect: connection refused
    2020/01/12 18:14:10 [INFO] memberlist: Marking 192.168.203.133 as failed, suspect timeout reached (0 peer confirmations)
    2020/01/12 18:14:10 [INFO] serf: EventMemberFailed: 192.168.203.133 192.168.203.133
    2020/01/12 18:14:10 [INFO] consul: Removing LAN server 192.168.203.133 (Addr: tcp/192.168.203.133:8300) (DC: dc1)
    2020/01/12 18:14:11 [INFO] memberlist: Suspect 192.168.203.133 has failed, no acks received
2020/01/12 18:14:16 [DEBUG] raft-net: 192.168.203.135:8300 accepted connection from: 192.168.203.134:48081
    2020/01/12 18:14:16 [INFO] raft: Duplicate RequestVote for same term: 3
    2020/01/12 18:14:18 [INFO] memberlist: Suspect 192.168.203.133.dc1 has failed, no acks received
    2020/01/12 18:14:18 [WARN] raft: Election timeout reached, restarting election
    2020/01/12 18:14:18 [INFO] raft: Node at 192.168.203.135:8300 [Candidate] entering Candidate state in term 4
2020/01/12 18:14:18 [WARN] Unable to get address for server id 57965f19-ecd6-eb1f-ad86-6993242862c0, using fallback address 192.168.203.133:8300: Could not find address for server id 57965f19-ecd6-eb1f-ad86-6993242862c0
    2020/01/12 18:14:18 [ERR] raft: Failed to make RequestVote RPC to {Voter 57965f19-ecd6-eb1f-ad86-6993242862c0 192.168.203.133:8300}: dial tcp 192.168.203.135:0->192.168.203.133:8300: connect: connection refused
    2020/01/12 18:14:18 [INFO] raft: Election won. Tally: 2
    2020/01/12 18:14:18 [INFO] raft: Node at 192.168.203.135:8300 [Leader] entering Leader state
    2020/01/12 18:14:18 [INFO] raft: Added peer 57965f19-ecd6-eb1f-ad86-6993242862c0, starting replication
    2020/01/12 18:14:18 [INFO] raft: Added peer 580d0fdf-dc4b-6d2a-ae97-4e4664cbb49f, starting replication
    2020/01/12 18:14:18 [INFO] consul: cluster leadership acquired
    2020/01/12 18:14:18 [INFO] consul: New leader elected: 192.168.203.135
2020/01/12 18:14:18 [WARN] Unable to get address for server id 57965f19-ecd6-eb1f-ad86-6993242862c0, using fallback address 192.168.203.133:8300: Could not find address for server id 57965f19-ecd6-eb1f-ad86-6993242862c0
    2020/01/12 18:14:18 [ERR] raft: Failed to AppendEntries to {Voter 57965f19-ecd6-eb1f-ad86-6993242862c0 192.168.203.133:8300}: dial tcp 192.168.203.135:0->192.168.203.133:8300: connect: connection refused
    2020/01/12 18:14:18 [INFO] raft: pipelining replication to peer {Voter 580d0fdf-dc4b-6d2a-ae97-4e4664cbb49f 192.168.203.134:8300}
    2020/01/12 18:14:18 [INFO] consul: member '192.168.203.133' failed, marking health critical
2020/01/12 18:14:18 [WARN] Unable to get address for server id 57965f19-ecd6-eb1f-ad86-6993242862c0, using fallback address 192.168.203.133:8300: Could not find address for server id 57965f19-ecd6-eb1f-ad86-6993242862c0
    2020/01/12 18:14:18 [ERR] raft: Failed to AppendEntries to {Voter 57965f19-ecd6-eb1f-ad86-6993242862c0 192.168.203.133:8300}: dial tcp 192.168.203.135:0->192.168.203.133:8300: connect: connection refused
2020/01/12 18:14:18 [WARN] Unable to get address for server id 57965f19-ecd6-eb1f-ad86-6993242862c0, using fallback address 192.168.203.133:8300: Could not find address for server id 57965f19-ecd6-eb1f-ad86-6993242862c0
    2020/01/12 18:14:18 [ERR] raft: Failed to AppendEntries to {Voter 57965f19-ecd6-eb1f-ad86-6993242862c0 192.168.203.133:8300}: dial tcp 192.168.203.135:0->192.168.203.133:8300: connect: connection refused
2020/01/12 18:14:18 [WARN] Unable to get address for server id 57965f19-ecd6-eb1f-ad86-6993242862c0, using fallback address 192.168.203.133:8300: Could not find address for server id 57965f19-ecd6-eb1f-ad86-6993242862c0
    2020/01/12 18:14:18 [ERR] raft: Failed to AppendEntries to {Voter 57965f19-ecd6-eb1f-ad86-6993242862c0 192.168.203.133:8300}: dial tcp 192.168.203.135:0->192.168.203.133:8300: connect: connection refused
2020/01/12 18:14:18 [WARN] Unable to get address for server id 57965f19-ecd6-eb1f-ad86-6993242862c0, using fallback address 192.168.203.133:8300: Could not find address for server id 57965f19-ecd6-eb1f-ad86-6993242862c0
    2020/01/12 18:14:18 [ERR] raft: Failed to AppendEntries to {Voter 57965f19-ecd6-eb1f-ad86-6993242862c0 192.168.203.133:8300}: dial tcp 192.168.203.135:0->192.168.203.133:8300: connect: connection refused
2020/01/12 18:14:18 [WARN] Unable to get address for server id 57965f19-ecd6-eb1f-ad86-6993242862c0, using fallback address 192.168.203.133:8300: Could not find address for server id 57965f19-ecd6-eb1f-ad86-6993242862c0
    2020/01/12 18:14:18 [ERR] raft: Failed to AppendEntries to {Voter 57965f19-ecd6-eb1f-ad86-6993242862c0 192.168.203.133:8300}: dial tcp 192.168.203.135:0->192.168.203.133:8300: connect: connection refused
2020/01/12 18:14:19 [WARN] Unable to get address for server id 57965f19-ecd6-eb1f-ad86-6993242862c0, using fallback address 192.168.203.133:8300: Could not find address for server id 57965f19-ecd6-eb1f-ad86-6993242862c0
    2020/01/12 18:14:19 [ERR] raft: Failed to AppendEntries to {Voter 57965f19-ecd6-eb1f-ad86-6993242862c0 192.168.203.133:8300}: dial tcp 192.168.203.135:0->192.168.203.133:8300: connect: connection refused
2020/01/12 18:14:19 [WARN] Unable to get address for server id 57965f19-ecd6-eb1f-ad86-6993242862c0, using fallback address 192.168.203.133:8300: Could not find address for server id 57965f19-ecd6-eb1f-ad86-6993242862c0
    2020/01/12 18:14:19 [ERR] raft: Failed to heartbeat to 192.168.203.133:8300: dial tcp 192.168.203.135:0->192.168.203.133:8300: connect: connection refused
2020/01/12 18:14:19 [WARN] Unable to get address for server id 57965f19-ecd6-eb1f-ad86-6993242862c0, using fallback address 192.168.203.133:8300: Could not find address for server id 57965f19-ecd6-eb1f-ad86-6993242862c0
    2020/01/12 18:14:19 [ERR] raft: Failed to AppendEntries to {Voter 57965f19-ecd6-eb1f-ad86-6993242862c0 192.168.203.133:8300}: dial tcp 192.168.203.135:0->192.168.203.133:8300: connect: connection refused
2020/01/12 18:14:20 [WARN] Unable to get address for server id 57965f19-ecd6-eb1f-ad86-6993242862c0, using fallback address 192.168.203.133:8300: Could not find address for server id 57965f19-ecd6-eb1f-ad86-6993242862c0
    2020/01/12 18:14:20 [ERR] raft: Failed to AppendEntries to {Voter 57965f19-ecd6-eb1f-ad86-6993242862c0 192.168.203.133:8300}: dial tcp 192.168.203.135:0->192.168.203.133:8300: connect: connection refused
2020/01/12 18:14:20 [WARN] Unable to get address for server id 57965f19-ecd6-eb1f-ad86-6993242862c0, using fallback address 192.168.203.133:8300: Could not find address for server id 57965f19-ecd6-eb1f-ad86-6993242862c0
    2020/01/12 18:14:20 [ERR] raft: Failed to heartbeat to 192.168.203.133:8300: dial tcp 192.168.203.135:0->192.168.203.133:8300: connect: connection refused
    2020/01/12 18:14:20 [WARN] consul: error getting server health from "192.168.203.133": rpc error getting client: failed to get conn: dial tcp 192.168.203.135:0->192.168.203.133:8300: connect: connection refused
    2020/01/12 18:14:21 [WARN] raft: Failed to contact 57965f19-ecd6-eb1f-ad86-6993242862c0 in 2.502407085s
2020/01/12 18:14:21 [WARN] Unable to get address for server id 57965f19-ecd6-eb1f-ad86-6993242862c0, using fallback address 192.168.203.133:8300: Could not find address for server id 57965f19-ecd6-eb1f-ad86-6993242862c0
    2020/01/12 18:14:21 [ERR] raft: Failed to heartbeat to 192.168.203.133:8300: dial tcp 192.168.203.135:0->192.168.203.133:8300: connect: connection refused
    2020/01/12 18:14:21 [WARN] consul: error getting server health from "192.168.203.133": context deadline exceeded
    2020/01/12 18:14:21 [WARN] consul: error getting server health from "192.168.203.134": context deadline exceeded
2020/01/12 18:14:21 [WARN] Unable to get address for server id 57965f19-ecd6-eb1f-ad86-6993242862c0, using fallback address 192.168.203.133:8300: Could not find address for server id 57965f19-ecd6-eb1f-ad86-6993242862c0
    2020/01/12 18:14:21 [ERR] raft: Failed to AppendEntries to {Voter 57965f19-ecd6-eb1f-ad86-6993242862c0 192.168.203.133:8300}: dial tcp 192.168.203.135:0->192.168.203.133:8300: connect: connection refused
2020/01/12 18:14:22 [WARN] Unable to get address for server id 57965f19-ecd6-eb1f-ad86-6993242862c0, using fallback address 192.168.203.133:8300: Could not find address for server id 57965f19-ecd6-eb1f-ad86-6993242862c0
    2020/01/12 18:14:22 [ERR] raft: Failed to heartbeat to 192.168.203.133:8300: dial tcp 192.168.203.135:0->192.168.203.133:8300: connect: connection refused
    2020/01/12 18:14:22 [WARN] consul: error getting server health from "192.168.203.133": rpc error getting client: failed to get conn: dial tcp 192.168.203.135:0->192.168.203.133:8300: connect: connection refused
2020/01/12 18:14:22 [WARN] Unable to get address for server id 57965f19-ecd6-eb1f-ad86-6993242862c0, using fallback address 192.168.203.133:8300: Could not find address for server id 57965f19-ecd6-eb1f-ad86-6993242862c0
    2020/01/12 18:14:22 [ERR] raft: Failed to heartbeat to 192.168.203.133:8300: dial tcp 192.168.203.135:0->192.168.203.133:8300: connect: connection refused
    2020/01/12 18:14:23 [INFO] serf: attempting reconnect to 192.168.203.133 192.168.203.133:8301
    2020/01/12 18:14:23 [WARN] raft: Failed to contact 57965f19-ecd6-eb1f-ad86-6993242862c0 in 4.932514106s
    2020/01/12 18:14:23 [WARN] consul: error getting server health from "192.168.203.133": context deadline exceeded
    2020/01/12 18:14:23 [WARN] consul: error getting server health from "192.168.203.134": context deadline exceeded
2020/01/12 18:14:23 [WARN] Unable to get address for server id 57965f19-ecd6-eb1f-ad86-6993242862c0, using fallback address 192.168.203.133:8300: Could not find address for server id 57965f19-ecd6-eb1f-ad86-6993242862c0
    2020/01/12 18:14:23 [ERR] raft: Failed to heartbeat to 192.168.203.133:8300: dial tcp 192.168.203.135:0->192.168.203.133:8300: connect: connection refused
2020/01/12 18:14:24 [WARN] Unable to get address for server id 57965f19-ecd6-eb1f-ad86-6993242862c0, using fallback address 192.168.203.133:8300: Could not find address for server id 57965f19-ecd6-eb1f-ad86-6993242862c0
    2020/01/12 18:14:24 [ERR] raft: Failed to AppendEntries to {Voter 57965f19-ecd6-eb1f-ad86-6993242862c0 192.168.203.133:8300}: dial tcp 192.168.203.135:0->192.168.203.133:8300: connect: connection refused
    2020/01/12 18:14:24 [WARN] consul: error getting server health from "192.168.203.133": rpc error getting client: failed to get conn: dial tcp 192.168.203.135:0->192.168.203.133:8300: connect: connection refused
2020/01/12 18:14:24 [WARN] Unable to get address for server id 57965f19-ecd6-eb1f-ad86-6993242862c0, using fallback address 192.168.203.133:8300: Could not find address for server id 57965f19-ecd6-eb1f-ad86-6993242862c0
    2020/01/12 18:14:24 [ERR] raft: Failed to heartbeat to 192.168.203.133:8300: dial tcp 192.168.203.135:0->192.168.203.133:8300: connect: connection refused
2020/01/12 18:14:25 [WARN] Unable to get address for server id 57965f19-ecd6-eb1f-ad86-6993242862c0, using fallback address 192.168.203.133:8300: Could not find address for server id 57965f19-ecd6-eb1f-ad86-6993242862c0
    2020/01/12 18:14:25 [ERR] raft: Failed to heartbeat to 192.168.203.133:8300: dial tcp 192.168.203.135:0->192.168.203.133:8300: connect: connection refused
    2020/01/12 18:14:25 [WARN] consul: error getting server health from "192.168.203.133": context deadline exceeded
    2020/01/12 18:14:25 [WARN] raft: Failed to contact 57965f19-ecd6-eb1f-ad86-6993242862c0 in 7.426744131s
    2020/01/12 18:14:26 [WARN] consul: error getting server health from "192.168.203.133": rpc error getting client: failed to get conn: dial tcp 192.168.203.135:0->192.168.203.133:8300: connect: connection refused
2020/01/12 18:14:26 [WARN] Unable to get address for server id 57965f19-ecd6-eb1f-ad86-6993242862c0, using fallback address 192.168.203.133:8300: Could not find address for server id 57965f19-ecd6-eb1f-ad86-6993242862c0
    2020/01/12 18:14:26 [ERR] raft: Failed to heartbeat to 192.168.203.133:8300: dial tcp 192.168.203.135:0->192.168.203.133:8300: connect: connection refused
    2020/01/12 18:14:27 [WARN] consul: error getting server health from "192.168.203.133": context deadline exceeded
    2020/01/12 18:14:28 [INFO] memberlist: Suspect 192.168.203.133.dc1 has failed, no acks received
    2020/01/12 18:14:28 [WARN] consul: error getting server health from "192.168.203.133": rpc error getting client: failed to get conn: dial tcp 192.168.203.135:0->192.168.203.133:8300: connect: connection refused
2020/01/12 18:14:28 [WARN] Unable to get address for server id 57965f19-ecd6-eb1f-ad86-6993242862c0, using fallback address 192.168.203.133:8300: Could not find address for server id 57965f19-ecd6-eb1f-ad86-6993242862c0
    2020/01/12 18:14:28 [ERR] raft: Failed to heartbeat to 192.168.203.133:8300: dial tcp 192.168.203.135:0->192.168.203.133:8300: connect: connection refused
2020/01/12 18:14:29 [WARN] Unable to get address for server id 57965f19-ecd6-eb1f-ad86-6993242862c0, using fallback address 192.168.203.133:8300: Could not find address for server id 57965f19-ecd6-eb1f-ad86-6993242862c0
    2020/01/12 18:14:29 [ERR] raft: Failed to AppendEntries to {Voter 57965f19-ecd6-eb1f-ad86-6993242862c0 192.168.203.133:8300}: dial tcp 192.168.203.135:0->192.168.203.133:8300: connect: connection refused
    2020/01/12 18:14:29 [WARN] consul: error getting server health from "192.168.203.133": context deadline exceeded
    2020/01/12 18:14:29 [WARN] consul: error getting server health from "192.168.203.134": context deadline exceeded
    2020/01/12 18:14:30 [WARN] consul: error getting server health from "192.168.203.133": rpc error getting client: failed to get conn: dial tcp 192.168.203.135:0->192.168.203.133:8300: connect: connection refused
    2020/01/12 18:14:31 [WARN] consul: error getting server health from "192.168.203.133": context deadline exceeded
    2020/01/12 18:14:31 [WARN] consul: error getting server health from "192.168.203.134": context deadline exceeded
2020/01/12 18:14:32 [WARN] Unable to get address for server id 57965f19-ecd6-eb1f-ad86-6993242862c0, using fallback address 192.168.203.133:8300: Could not find address for server id 57965f19-ecd6-eb1f-ad86-6993242862c0
    2020/01/12 18:14:32 [ERR] raft: Failed to heartbeat to 192.168.203.133:8300: dial tcp 192.168.203.135:0->192.168.203.133:8300: connect: connection refused
    2020/01/12 18:14:32 [WARN] consul: error getting server health from "192.168.203.133": rpc error getting client: failed to get conn: dial tcp 192.168.203.135:0->192.168.203.133:8300: connect: connection refused
    2020/01/12 18:14:33 [WARN] consul: error getting server health from "192.168.203.133": context deadline exceeded
    2020/01/12 18:14:34 [WARN] consul: error getting server health from "192.168.203.133": rpc error getting client: failed to get conn: dial tcp 192.168.203.135:0->192.168.203.133:8300: connect: connection refused
    2020/01/12 18:14:35 [WARN] consul: error getting server health from "192.168.203.133": context deadline exceeded
    2020/01/12 18:14:36 [WARN] consul: error getting server health from "192.168.203.133": rpc error getting client: failed to get conn: dial tcp 192.168.203.135:0->192.168.203.133:8300: connect: connection refused
    2020/01/12 18:14:37 [WARN] consul: error getting server health from "192.168.203.133": context deadline exceeded
2020/01/12 18:14:38 [WARN] Unable to get address for server id 57965f19-ecd6-eb1f-ad86-6993242862c0, using fallback address 192.168.203.133:8300: Could not find address for server id 57965f19-ecd6-eb1f-ad86-6993242862c0
    2020/01/12 18:14:38 [ERR] raft: Failed to heartbeat to 192.168.203.133:8300: dial tcp 192.168.203.135:0->192.168.203.133:8300: connect: connection refused
    2020/01/12 18:14:38 [INFO] memberlist: Suspect 192.168.203.133.dc1 has failed, no acks received
    2020/01/12 18:14:38 [WARN] consul: error getting server health from "192.168.203.133": rpc error getting client: failed to get conn: dial tcp 192.168.203.135:0->192.168.203.133:8300: connect: connection refused
    2020/01/12 18:14:39 [WARN] consul: error getting server health from "192.168.203.133": context deadline exceeded
    2020/01/12 18:14:39 [WARN] consul: error getting server health from "192.168.203.134": context deadline exceeded
2020/01/12 18:14:39 [WARN] Unable to get address for server id 57965f19-ecd6-eb1f-ad86-6993242862c0, using fallback address 192.168.203.133:8300: Could not find address for server id 57965f19-ecd6-eb1f-ad86-6993242862c0
    2020/01/12 18:14:39 [ERR] raft: Failed to AppendEntries to {Voter 57965f19-ecd6-eb1f-ad86-6993242862c0 192.168.203.133:8300}: dial tcp 192.168.203.135:0->192.168.203.133:8300: connect: connection refused
    2020/01/12 18:14:40 [INFO] memberlist: Marking 192.168.203.133.dc1 as failed, suspect timeout reached (0 peer confirmations)
    2020/01/12 18:14:40 [INFO] serf: EventMemberFailed: 192.168.203.133.dc1 192.168.203.133
    2020/01/12 18:14:40 [INFO] consul: Handled member-failed event for server "192.168.203.133.dc1" in area "wan"
    2020/01/12 18:14:40 [WARN] consul: error getting server health from "192.168.203.133": rpc error getting client: failed to get conn: dial tcp 192.168.203.135:0->192.168.203.133:8300: connect: connection refused
    2020/01/12 18:14:41 [WARN] consul: error getting server health from "192.168.203.133": context deadline exceeded
    2020/01/12 18:14:41 [WARN] consul: error getting server health from "192.168.203.134": context deadline exceeded
    2020/01/12 18:14:42 [WARN] consul: error getting server health from "192.168.203.133": rpc error getting client: failed to get conn: dial tcp 192.168.203.135:0->192.168.203.133:8300: connect: connection refused
    2020/01/12 18:14:43 [WARN] consul: error getting server health from "192.168.203.133": context deadline exceeded
    2020/01/12 18:14:44 [WARN] consul: error getting server health from "192.168.203.133": rpc error getting client: failed to get conn: dial tcp 192.168.203.135:0->192.168.203.133:8300: connect: connection refused
    2020/01/12 18:14:45 [WARN] consul: error getting server health from "192.168.203.133": context deadline exceeded
    2020/01/12 18:14:46 [WARN] consul: error getting server health from "192.168.203.133": rpc error getting client: failed to get conn: dial tcp 192.168.203.135:0->192.168.203.133:8300: connect: connection refused
    2020/01/12 18:14:47 [WARN] consul: error getting server health from "192.168.203.133": context deadline exceeded
    2020/01/12 18:14:48 [WARN] consul: error getting server health from "192.168.203.133": rpc error getting client: failed to get conn: dial tcp 192.168.203.135:0->192.168.203.133:8300: connect: connection refused
2020/01/12 18:14:49 [WARN] Unable to get address for server id 57965f19-ecd6-eb1f-ad86-6993242862c0, using fallback address 192.168.203.133:8300: Could not find address for server id 57965f19-ecd6-eb1f-ad86-6993242862c0
    2020/01/12 18:14:49 [ERR] raft: Failed to heartbeat to 192.168.203.133:8300: dial tcp 192.168.203.135:0->192.168.203.133:8300: connect: connection refused
    2020/01/12 18:14:49 [WARN] consul: error getting server health from "192.168.203.133": context deadline exceeded
2020/01/12 18:14:49 [WARN] Unable to get address for server id 57965f19-ecd6-eb1f-ad86-6993242862c0, using fallback address 192.168.203.133:8300: Could not find address for server id 57965f19-ecd6-eb1f-ad86-6993242862c0
    2020/01/12 18:14:49 [ERR] raft: Failed to AppendEntries to {Voter 57965f19-ecd6-eb1f-ad86-6993242862c0 192.168.203.133:8300}: dial tcp 192.168.203.135:0->192.168.203.133:8300: connect: connection refused
    2020/01/12 18:14:50 [WARN] consul: error getting server health from "192.168.203.133": rpc error getting client: failed to get conn: dial tcp 192.168.203.135:0->192.168.203.133:8300: connect: connection refused
    2020/01/12 18:14:51 [WARN] consul: error getting server health from "192.168.203.133": context deadline exceeded
    2020/01/12 18:14:51 [WARN] consul: error getting server health from "192.168.203.134": context deadline exceeded
    2020/01/12 18:14:52 [WARN] consul: error getting server health from "192.168.203.133": rpc error getting client: failed to get conn: dial tcp 192.168.203.135:0->192.168.203.133:8300: connect: connection refused
    2020/01/12 18:14:53 [WARN] consul: error getting server health from "192.168.203.133": context deadline exceeded
    2020/01/12 18:14:53 [WARN] consul: error getting server health from "192.168.203.134": context deadline exceeded
    2020/01/12 18:14:54 [WARN] consul: error getting server health from "192.168.203.133": rpc error getting client: failed to get conn: dial tcp 192.168.203.135:0->192.168.203.133:8300: connect: connection refused
    2020/01/12 18:14:55 [WARN] consul: error getting server health from "192.168.203.133": context deadline exceeded
    2020/01/12 18:14:55 [WARN] consul: error getting server health from "192.168.203.134": context deadline exceeded
    2020/01/12 18:14:56 [WARN] consul: error getting server health from "192.168.203.133": rpc error getting client: failed to get conn: dial tcp 192.168.203.135:0->192.168.203.133:8300: connect: connection refused
    2020/01/12 18:14:57 [WARN] consul: error getting server health from "192.168.203.133": context deadline exceeded
    2020/01/12 18:14:57 [WARN] consul: error getting server health from "192.168.203.134": context deadline exceeded
    2020/01/12 18:14:58 [WARN] consul: error getting server health from "192.168.203.133": rpc error getting client: failed to get conn: dial tcp 192.168.203.135:0->192.168.203.133:8300: connect: connection refused
    2020/01/12 18:14:59 [WARN] consul: error getting server health from "192.168.203.133": context deadline exceeded
    2020/01/12 18:14:59 [WARN] consul: error getting server health from "192.168.203.134": context deadline exceeded
2020/01/12 18:14:59 [WARN] Unable to get address for server id 57965f19-ecd6-eb1f-ad86-6993242862c0, using fallback address 192.168.203.133:8300: Could not find address for server id 57965f19-ecd6-eb1f-ad86-6993242862c0
    2020/01/12 18:14:59 [ERR] raft: Failed to heartbeat to 192.168.203.133:8300: dial tcp 192.168.203.135:0->192.168.203.133:8300: connect: connection refused
2020/01/12 18:15:00 [WARN] Unable to get address for server id 57965f19-ecd6-eb1f-ad86-6993242862c0, using fallback address 192.168.203.133:8300: Could not find address for server id 57965f19-ecd6-eb1f-ad86-6993242862c0
    2020/01/12 18:15:00 [ERR] raft: Failed to AppendEntries to {Voter 57965f19-ecd6-eb1f-ad86-6993242862c0 192.168.203.133:8300}: dial tcp 192.168.203.135:0->192.168.203.133:8300: connect: connection refused
    2020/01/12 18:15:00 [WARN] consul: error getting server health from "192.168.203.133": rpc error getting client: failed to get conn: dial tcp 192.168.203.135:0->192.168.203.133:8300: connect: connection refused
    2020/01/12 18:15:01 [WARN] consul: error getting server health from "192.168.203.133": context deadline exceeded
    2020/01/12 18:15:01 [WARN] consul: error getting server health from "192.168.203.134": context deadline exceeded
    2020/01/12 18:15:01 [INFO] serf: EventMemberJoin: 192.168.203.133 192.168.203.133
    2020/01/12 18:15:01 [INFO] consul: Adding LAN server 192.168.203.133 (Addr: tcp/192.168.203.133:8300) (DC: dc1)
    2020/01/12 18:15:01 [INFO] consul: member '192.168.203.133' joined, marking health alive
    2020/01/12 18:15:02 [INFO] serf: EventMemberJoin: 192.168.203.133.dc1 192.168.203.133
    2020/01/12 18:15:02 [INFO] consul: Handled member-join event for server "192.168.203.133.dc1" in area "wan"
2020/01/12 18:15:08 [DEBUG] raft-net: 192.168.203.135:8300 accepted connection from: 192.168.203.133:54216
    2020/01/12 18:15:08 [WARN] raft: Rejecting vote request from 192.168.203.133:8300 since we have a leader: 192.168.203.135:8300
    2020/01/12 18:15:10 [INFO] raft: pipelining replication to peer {Voter 57965f19-ecd6-eb1f-ad86-6993242862c0 192.168.203.133:8300}
```








### 4. 效果图如下

![1578824157274](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\1578824157274.png)



### 5. 查看集群节点

```shell
[root@master zhangwei]# ./consul members
Node             Address               Status  Type    Build  Protocol  DC   Segment
192.168.203.133  192.168.203.133:8301  alive   server  1.2.0  2         dc1  <all>
192.168.203.134  192.168.203.134:8301  alive   server  1.2.0  2         dc1  <all>
192.168.203.135  192.168.203.135:8301  alive   server  1.2.0  2         dc1  <all>

[root@master zhangwei]# ./consul operator raft list-peers
Node             ID                                    Address               State     Voter  RaftProtocol
192.168.203.133  57965f19-ecd6-eb1f-ad86-6993242862c0  192.168.203.133:8300  leader    true   3
192.168.203.134  580d0fdf-dc4b-6d2a-ae97-4e4664cbb49f  192.168.203.134:8300  follower  true   3
192.168.203.135  a8b1440a-1ce9-9e72-9e08-717f9170e4eb  192.168.203.135:8300  follower  true   3
```





## 总结

> 以上仅为本人实操记录文档
>
> 这里有更全资料：  https://www.cnblogs.com/jpfss/p/11316667.html 