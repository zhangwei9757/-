# Spring Cloud 微服务系列三：配置中心 （整合bus,rabbitMQ实现配置文件动态刷新）



## 个人提示： 如果 consul ,rabbitMQ 不是很明白，可以先阅读本公众号相关文章

> ​        传统web项目，配置文件基于项目本身或者启动时配置进去，耦合性太强，不利于动态
>
> 管理当使用微服务架构时，面临了配置文件不可单一独立配置的问题，而spring cloud提出了
>
> 解决方案，使用配置中心，当服务启动时，或者运行中需要修改部分配置或者参数时，不用
>
> 重启甚至于不用修改任务代码，无感动态修改。下面基于 config组件的使用与详解。
>
> ​		上一篇提到注册中心， 本篇配置中心基于注册中心，一旦有客户端加入，从注册中心获取
>
> 配置中心注册地址即: ip:port 拉取配置好的文件信息，加载到本工程模块



## 配置中心架构

> 1. 配置文件存放， 即：配置文件     
>
>    ​	基于 github:   https://github.com/zhangwei9757/mirco-service-configrations
>
> 2. 配置中心应用， 即：配置中心服务器
>
>    ​    config-server
>
> 3. 访问配置中心的客户端， 即：客户端
>
>    ​	oonfig-client



## config-server 搭建


### 1. 使用IDEA 创建一个配置服务端应用

![config-server-init](https://github.com/zhangwei9757/Documents/blob/master/images/config-server-init.png?raw=true)



### 2. 添加依赖 :  spring-cloud-config-server

```xml
<!-- 配置中心 -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-server</artifactId>
</dependency>

<!-- 服务发现 -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-consul-discovery</artifactId>
</dependency>

<!-- bus -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bus-amqp</artifactId>
</dependency>
```

```yam
server:
  port: 9000
spring:
  cloud:
    consul:
      host: 192.168.203.133
      port: 8500
      enabled: true
      discovery:
        enabled: true
        hostname: localhost
        instance-id: ${spring.application.name}:${spring.cloud.consul.discovery.hostname}
        service-name: ${spring.application.name}
        healthCheckInterval: 15s
        health-check-critical-timeout: 60s
        tags: dev
        heartbeat:
          enabled: true
    config:  # 开启配置中心，加载远程github仓库目录文件资源
      server:
        git:
          uri: https://github.com/zhangwei9757/mirco-service-configrations.git
    bus:
      trace:
        enabled: true # 开启消息总线 & bus-refresh 端口
      enabled: true
      
management:
  endpoints:
    web:
      exposure:
        include: "*" #暴露所有节点 "bus-refresh"
  endpoint:
    health:
      show-details: ALWAYS  #显示详细信息
```



### 3. 启动项目 & 测试

```java
@SpringBootApplication
@EnableConfigServer
@EnableDiscoveryClient
@Slf4j
// 右键直接运行
public class ConfigServerApplication {

    public static void main(String[] args) {
        ServletWebServerApplicationContext swa = (ServletWebServerApplicationContext) SpringApplication.run(ConfigServerApplication.class, args);
        WebServer server = swa.getWebServer();
        if (server != null) {
            log.warn("\n\r --->>>Configration Service, 服务监听端口:【" + server.getPort() + "】, day:【" + TimeUtils.getCurrentDay() + "】");
        }
    }
}
```



![config-server-consul](https://github.com/zhangwei9757/Documents/blob/master/images/config-server-consul.png?raw=true)

5. URL与配置文件的映射关系如下：

   ```txt
   /{application}/{profile}[/{label}]
   /{application}-{profile}.yml
   /{label}/{application}-{profile}.yml
   /{application}-{profile}.properties
   /{label}/{application}-{profile}.properties
   ```

6. 测试结果如下图  【客户端配置文件信息】：

   ![1579336637693](https://github.com/zhangwei9757/Documents/blob/master/images/uc.yml.png?raw=true)

   
   
   ```yml
   feign:
     hystrix:
       enabled: true
   hystrix:
     command:
       default:
         execution:
           isolation:
             thread:
               timeoutInMilliseconds: 15000
           timeout: ''
   management:
     endpoint:
       health:
         show-details: ALWAYS
     endpoints:
       health:
         senysssitive: false
       web:
         exposure:
           include: '*'
   ribbon:
     ConnectTimeout: 6000
     MaxAutoRetries: 0
     MaxAutoRetriesNextServer: 0
     ReadTimeout: 6000
   server:
     port: 9001
   spring:
     application:
       name: gateway-service
     boot:
       admin:
         client:
           instance:
             prefer-ip: true
             service-url: http://${spring.cloud.client.ip-address}:9001
           url: http://localhost:9002
     bus:
       enabled: true
       trace:
         enabled: true
     cloud:
       gateway:
         discovery:
           locator:
             enabled: true
             lower-case-service-id: true
         routes:
         - filters:
           - RewritePath=/zz/(?<segment>.*), /$\{segment}
           id: rewrite_to_devops
           predicates:
           - Path=/zz/**
           uri: lb://devops-provider
         - filters:
           - RewritePath=/zz/(?<segment>.*), /$\{segment}
           id: rewrite_to_devops
           predicates:
           - Path=/zz/**
           uri: lb://devops-provider
         - filters:
           - StripPrefix=1
           id: service_to_baidu
           predicates:
           - Path=/baidu/**
           uri: http://www.baidu.com
     profiles:
       active:
       - dev
     rabbitmq:
       host: localhost
       password: guest
       port: 5672
       username: guest
     sleuth:
       sampler:
         probability: 1
       web:
         client:
           enabled: true
     zipkin:
    base-url: http://49.232.32.242:9411/
   ```
   
   

###  4. EnableConfigServer 注解

@EnableConfigServer ： 注解自动开启动配置中心服务器

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import({ConfigServerConfiguration.class})
public @interface EnableConfigServer {
}



@Configuration
public class ConfigServerConfiguration {
    public ConfigServerConfiguration() {
    }

    @Bean
    public ConfigServerConfiguration.Marker enableConfigServerMarker() {
        return new ConfigServerConfiguration.Marker();
    }

    class Marker {
        Marker() {
        }
    }
}

@Configuration
@ConditionalOnBean({Marker.class})
@EnableConfigurationProperties({ConfigServerProperties.class})
@Import({EnvironmentRepositoryConfiguration.class, CompositeConfiguration.class, ResourceRepositoryConfiguration.class, ConfigServerEncryptionConfiguration.class, ConfigServerMvcConfiguration.class})
public class ConfigServerAutoConfiguration {
    public ConfigServerAutoConfiguration() {
    }
}
```

通过注解源码可见

1. 当我们开启动 @EnableConfigServer 时会自动 引入 ConfigServerConfiguration配置类

2. ConfigServerConfiguration会自动注入 Marker

3. ConfigServerAutoConfiguration配置类，监测到 Marker的存在时 @ConditionalOnBean({Marker.class})

   会自动引入其它配置相关类:

   EnvironmentRepositoryConfiguration： 			环境变量存储相关的配置类
   CompositeConfiguration：								 组合方式的环境仓库配置类
   ResourceRepositoryConfiguration：				  资源仓库相关的配置类
   ConfigServerEncryptionConfiguration：             加密断点相关的配置类
   ConfigServerMvcConfiguration：                       对外暴露的MVC端点控制器的配置类

4.  以上配置类具体功能可以打开源码查看，本人只对其中一个展开解析

   ###  EnvironmentRepositoryConfiguration 是环境变量存储相关的配置类，它本身也提供了很多实现 

   ```java
   @Configuration
   @Profile({"git"})
   class GitRepositoryConfiguration extends DefaultRepositoryConfiguration {
       GitRepositoryConfiguration() {
       }
   }
   
   @Configuration
   @ConditionalOnMissingBean(
       value = {EnvironmentRepository.class}, // 当前容器没有环境配置时，默认使用 Git
       search = SearchStrategy.CURRENT
   )
   class DefaultRepositoryConfiguration {
       @Autowired
       private ConfigurableEnvironment environment;
       @Autowired
       private ConfigServerProperties server;
       @Autowired(
           required = false
       )
       private TransportConfigCallback customTransportConfigCallback;
   
       DefaultRepositoryConfiguration() {
       }
   
       @Bean
       public MultipleJGitEnvironmentRepository defaultEnvironmentRepository(MultipleJGitEnvironmentRepositoryFactory gitEnvironmentRepositoryFactory, MultipleJGitEnvironmentProperties environmentProperties) throws Exception {
           return gitEnvironmentRepositoryFactory.build(environmentProperties);
       }
   }
   ```

   

   5. 类图如下

   ![MultipleJGitEnvironmentRepository](https://github.com/zhangwei9757/Documents/blob/master/images/MultipleJGitEnvironmentRepository.png?raw=true)

   
   
   总结： 我们可以看见，默认使用git 实现



## config-client 搭建

### 1. 使用IDEA 创建一个配置客户端应用    (新建方法同服务端，只是应用名修改为客户端即可)

### 2. 添加依赖：spring-cloud-starter-config

```xml
<!-- 客户端配置 -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-config</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-consul-discovery</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bus-amqp</artifactId>
</dependency>
```

```yml
spring:
  cloud:
    consul:
      host: localhost
      port: 8500
      discovery:
        enabled: true
        hostname: localhost
        instance-id: ${spring.application.name}:${spring.cloud.consul.discovery.hostname}
        service-name: ${spring.application.name}
        health-check-path: /actuator/health
        healthCheckInterval: 15s
        health-check-critical-timeout: 60s
        tags: dev
        heartbeat:
          enabled: true
    config:
      name: gateway
      profile: dev
      label: master
      fail-fast: true
      discovery:
        enabled: true
        service-id: config-service

```



### 3. 启动项目 & 测试

```java
@SpringBootApplication
@EnableDiscoveryClient
@Slf4j
public class UcServerApplication {

    public static void main(String[] args) {
        ConfigurableApplicationContext swa = SpringApplication.run(UcServerApplication.class, args);
        ConfigurableEnvironment environment = swa.getEnvironment();
        if (environment != null) {
            String port = environment.getProperty("server.port");
            log.warn("\n\r --->>> UC Service Listener Port:【" + port + "】, day:【" + TimeUtils.getCurrentDay() + "】");
        }
    }
}
```

![1579336227844](https://github.com/zhangwei9757/Documents/blob/master/images/config-client-init.png?raw=true)



1. 编写测试controller

```java
package com.microservice.controller;

import com.microservice.dto.basic.JsonResult;
import com.microservice.feigin.TurbineFeignService;
import com.microservice.feigin.UploadService;
import com.microservice.security.MicroRedisUserDetailsServiceImpl;
import com.microservice.security.UserDetailsServiceImpl;
import com.microservice.utils.JsonUtils;
import io.swagger.annotations.*;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.cloud.context.config.annotation.RefreshScope;
import org.springframework.http.HttpStatus;
import org.springframework.http.MediaType;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.multipart.MultipartFile;

@RestController
@RequestMapping("/api")
@Slf4j
@RefreshScope
public class ApiController {

    @Value("${test}")
    String test;

    @GetMapping("/index")
    public String index() {
        return "uc-server index info....."+ " test: " + test;
    }
}

```



2. 测试结果如下图：

![1579336486997](https://github.com/zhangwei9757/Documents/blob/master/images/test.png?raw=true)

3. 修改 github文件键值对为新值：  test:zhangwei222

4. 调用客户端端口:  /actuator/bus-refresh 

![1579336935224](https://github.com/zhangwei9757/Documents/blob/master/images/config-client-refresh.png?raw=true)

5. 调用测试2步骤

![1579336881459](https://github.com/zhangwei9757/Documents/blob/master/images/test2.png?raw=true)



##  个人总结：

> 1.  微服务入门前置技能： springboot 搭建maven工程 添加依赖
> 2.  上篇注册中心，实践操作并配置好环境
> 3.  本篇配置中心，实践操作并配置好环境 





## 持续更新中 ..... 