# Spring Cloud 微服务系列一 ：入门



 ![å¨è¿éæå¥å¾çæè¿°](https://img-blog.csdnimg.cn/20181229154825929.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2Z6eTYyOTQ0MjQ2Ng==,size_16,color_FFFFFF,t_70) 

>  微服务是一种架构风格，一个大型复杂软件应用由一个或多个微服务组成。系统中的各个微服务可被独立部署，各个微服务之间是松耦合的。每个微服务仅关注于完成一件任务并很好地完成该任务。在所有情况下，每个任务代表着一个小的业务能力。 
>
> Spring Cloud是一系列框架的有序集合。它利用Spring Boot的开发便利性巧妙地简化了分布式系统基础设施的开发，如服务发现注册、配置中心、消息总线、负载均衡、断路器、数据监控等，都可以用Spring Boot的开发风格做到一键启动和部署。Spring并没有重复制造轮子，它只是将目前各家公司开发的比较成熟、经得起实际考验的服务框架组合起来，通过Spring Boot风格进行再封装屏蔽掉了复杂的配置和实现原理，最终给开发者留出了一套简单易懂、易部署和易维护的分布式系统开发工具包。
>
> Spring Cloud致力于为典型的用例和扩展机制提供良好的开箱即用体验，以涵盖其他用例。
>
> - 分布式/版本化配置
> - 服务注册和发现
> - 路由
> - 服务到服务的呼叫
> - 负载均衡
> - 断路器
> - 全局锁
> - 领导选举和集群状态
> - 分布式消息传递



## Spring Boot && Cloud 版本关系

> 来源官网：  https://spring.io/projects/spring-cloud 
>
> 国内中文文档：  https://www.springcloud.cc/ 

| Release Train | Boot Version |
| ------------- | ------------ |
| Hoxton        | 2.2.x        |
| Greenwich     | 2.1.x        |
| Finchley      | 2.0.x        |
| Edgware       | 1.5.x        |
| Dalston       | 1.5.x        |



| Component                 | Edgware.SR6    | Greenwich.SR2 | Greenwich.BUILD-SNAPSHOT |
| ------------------------- | -------------- | ------------- | ------------------------ |
| spring-cloud-aws          | 1.2.4.RELEASE  | 2.1.2.RELEASE | 2.1.3.BUILD-SNAPSHOT     |
| spring-cloud-bus          | 1.3.4.RELEASE  | 2.1.2.RELEASE | 2.1.3.BUILD-SNAPSHOT     |
| spring-cloud-cli          | 1.4.1.RELEASE  | 2.0.0.RELEASE | 2.0.1.BUILD-SNAPSHOT     |
| spring-cloud-commons      | 1.3.6.RELEASE  | 2.1.2.RELEASE | 2.1.3.BUILD-SNAPSHOT     |
| spring-cloud-contract     | 1.2.7.RELEASE  | 2.1.2.RELEASE | 2.1.3.BUILD-SNAPSHOT     |
| spring-cloud-config       | 1.4.7.RELEASE  | 2.1.3.RELEASE | 2.1.4.BUILD-SNAPSHOT     |
| spring-cloud-netflix      | 1.4.7.RELEASE  | 2.1.2.RELEASE | 2.1.3.BUILD-SNAPSHOT     |
| spring-cloud-security     | 1.2.4.RELEASE  | 2.1.3.RELEASE | 2.1.4.BUILD-SNAPSHOT     |
| spring-cloud-cloudfoundry | 1.1.3.RELEASE  | 2.1.2.RELEASE | 2.1.3.BUILD-SNAPSHOT     |
| spring-cloud-consul       | 1.3.6.RELEASE  | 2.1.2.RELEASE | 2.1.3.BUILD-SNAPSHOT     |
| spring-cloud-sleuth       | 1.3.6.RELEASE  | 2.1.1.RELEASE | 2.1.2.BUILD-SNAPSHOT     |
| spring-cloud-stream       | Ditmars.SR5    | Fishtown.SR3  | Fishtown.BUILD-SNAPSHOT  |
| spring-cloud-zookeeper    | 1.2.3.RELEASE  | 2.1.2.RELEASE | 2.1.3.BUILD-SNAPSHOT     |
| spring-boot               | 1.5.21.RELEASE | 2.1.5.RELEASE | 2.1.8.BUILD-SNAPSHOT     |
| spring-cloud-task         | 1.2.4.RELEASE  | 2.1.2.RELEASE | 2.1.3.BUILD-SNAPSHOT     |
| spring-cloud-vault        | 1.1.3.RELEASE  | 2.1.2.RELEASE | 2.1.3.BUILD-SNAPSHOT     |
| spring-cloud-gateway      | 1.0.3.RELEASE  | 2.1.2.RELEASE | 2.1.3.BUILD-SNAPSHOT     |
| spring-cloud-openfeign    |                | 2.1.2.RELEASE | 2.1.3.BUILD-SNAPSHOT     |
| spring-cloud-function     | 1.0.2.RELEASE  | 2.0.2.RELEASE | 2.0.3.BUILD-SNAPSHOT     |





## 子项目

Spring Cloud Config：配置管理开发工具包，可以让你把配置放到远程服务器，目前支持本地存储、Git以及Subversion。

Spring Cloud Bus：事件、消息总线，用于在集群（例如，配置变化事件）中传播状态变化，可与Spring Cloud Config联合实现热部署。

Spring Cloud Netflix：针对多种Netflix组件提供的开发工具包，其中包括Eureka、Hystrix、Zuul、Archaius等。

Netflix Eureka：云端负载均衡，一个基于 REST 的服务，用于定位服务，以实现云端的负载均衡和中间层服务器的故障转移。

Netflix Hystrix：容错管理工具，旨在通过控制服务和第三方库的节点,从而对延迟和故障提供更强大的容错能力。

Netflix Zuul：边缘服务工具，是提供动态路由，监控，弹性，安全等的边缘服务。

Netflix Archaius：配置管理API，包含一系列配置管理API，提供动态类型化属性、线程安全配置操作、轮询框架、回调机制等功能。

Spring Cloud for Cloud Foundry：通过Oauth2协议绑定服务到CloudFoundry，CloudFoundry是VMware推出的开源PaaS云平台。

Spring Cloud Sleuth：日志收集工具包，封装了Dapper,Zipkin和HTrace操作。

Spring Cloud Data Flow：大数据操作工具，通过命令行方式操作数据流。

Spring Cloud Security：安全工具包，为你的应用程序添加安全控制，主要是指OAuth2。

Spring Cloud Consul：封装了Consul操作，consul是一个服务发现与配置工具，与Docker容器可以无缝集成。

Spring Cloud Zookeeper：操作Zookeeper的工具包，用于使用zookeeper方式的服务注册和发现。

Spring Cloud Stream：数据流操作开发包，封装了与Redis,Rabbit、Kafka等发送接收消息。

Spring Cloud CLI：基于 Spring Boot CLI，可以让你以命令行方式快速建立云组件。





## 网易云案例架构

 ![å¨è¿éæå¥å¾çæè¿°](https://img-blog.csdnimg.cn/20181210203748541.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3pob3V0YW9jaHVu,size_16,color_FFFFFF,t_70) 

 ![å¨è¿éæå¥å¾çæè¿°](https://img-blog.csdnimg.cn/20181210203806775.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3pob3V0YW9jaHVu,size_16,color_FFFFFF,t_70) 





## 开发工具与环境对应版本

> jdk 1.8 +
>
> maven 3.0 +
>
> spring boot 2.0.x
>
> spring cloud Finchley



## 使用组件

	consul 集群注册中心
	
	config 配置中心 + git
	
	bus 消息总线 + rabbitMQ 
	
	gateway 第二代网关，与zuul相关性能有极大的提升
	
	security 安全组件，权限管理框架 替代shiro框架，使用更简洁，轻量
	
	openfeign 原feign升级版，以接口形式更优雅的调用其它微服模块 底层使用 RestTemplate
	
	ribbon 使openfeign调用实现负载均衡，自由配置策略
	
	zipkin链路追踪,分析 http请求调用情况
	
	admin 监控 springboot 项目启动基本信息 jvm thread ...
	
	turbine 豪猪集群监控 openfeign  api 调用性能监控
	
	......



## 持续更新中...

