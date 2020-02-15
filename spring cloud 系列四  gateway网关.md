# Spring Cloud 微服务系列四：gateway网关



## 1.  什么是 Spring Cloud Gateway

> Spring Cloud Gatey 是Spring 官方基于 Spring 5.0、Spring Boot 2.0 和 Project Reactor 等技术开发的网关， Spring Cloud Gateway 指在为微服务架构提供简单、有效且统一的 API路由管理方式， 作为Spring Cloud 生态系统中的网关，目标是替代 Netflix Zuul, 不仅提供统一的路由方式， 并且还基于Filter 链的方式提供了网关基本功能，如：安全、监控/埋点、限流等。其性能比原 Zuul 网关提高了50%。
>
> 注意：Spring Cloud Gateway 启动容器目前只支持 Netty，所以注意导包冲突问题。



## 2.  Spring Cloud Gateway 的核心概念

> 1. 路由 (route)。路由是网关最基础的部分，路由信息由一个ID、url、断言工厂和 Filter 组成 。 
>2. 断言 (predicate)。Java8中的断言函数。Spring Clouc Gateway 中的断言函数输入类是 Spring5.0 框架中的 ServerWebExchange。允许自定义匹配来自于 Http Request 中的任何信息，比如请求头和参数等。具体分类见下文中涉及的 'Spring Cloud Gateway 的路由断言 ' 部分。
> 3. 过滤器 (filter)。一个标准的 Spring webFilter。有二种类型分别是：Gateway Filter 和 Global Filter 。
>



## 3.  Spring Cloud Gateway 的工作原理 

> 1. 客户端会向 Gateway 发起请求，请求首先会被 Http WebHandlerAdapter 进行提取组装成网关的上下文
> 2. 然后网关上下文会传递到 DispatcherHandler。DispatcherHandler是所有请求的分发处理器，DispatcherHandler主要负责分发请求对应的处理器，比如将请求分发到对应RoutePredicateHandlerMapping（路由断言处理映射器）。
> 3. 路由断言处理映射器主要用于路由的查找，并返回对应的FilteringWebHandler
> 4. FilteringWebHandler主要负责组装 Filter链表并调用Filter处理，然后把请求转到后端对应的微服务处理
> 5. 处理完毕之后，Response返回到 Gateway 调用的客户端



![gateway_core](https://raw.githubusercontent.com/zhangwei9757/Documents/master/images/gateway_core.png)



## 4.  Spring Cloud Gateway 入门案例

> 作为网关来说，最重要的功能就是协议适配和协议转发，协议转发是基本的路由信息转发，演示案例如下

### 1.创建Maven 工程

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<!--Spring Cloud Gateway的Starter-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>
```



### 2. 编写主入口程序  (方式一)

```java
@SpringBootApplication
public class SpringCloudGatewayApplication {

	/**
	 * 基本的转发 实现自动跳转到京东
	 * 当访问http://localhost:8080/jd
	 * 转发到http://jd.com
	 * @param builder
	 * @return
	 */
	@Bean
	public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
		return builder.routes()
					.route(r ->r.path("/jd")											    							   .uri("http://jd.com:80/")
                       		 .id("jd_route")
					).build();
	}

	public static void main(String[] args) {
		SpringApplication.run(SpringCloudGatewayApplication.class, args);
	}
}
```



### 3.使用配置文件方式 （方式二）

```yml
server:
  port: 8080
spring:
  application:
    name: spring-cloud-gateway
  cloud:
    gateway:
      routes: #当访问http://localhost:8080/baidu,直接转发到https://www.baidu.com/
      - id: baidu_route
        uri: http://baidu.com:80/
        predicates:
        - Path=/baidu


logging: ## Spring Cloud Gateway的日志配置
  level:
    org.springframework.cloud.gateway: TRACE
    org.springframework.http.server.reactive: DEBUG
    org.springframework.web.reactive: DEBUG
    reactor.ipc.netty: DEBUG

management:
  endpoints:
    web:
      exposure:
        include: '*'
  security:
    enabled: false
```



### 4. Spring Cloud Gateway 提供了一个 gateway actuator， 该 EndPoint 提供了关于Filter 及 routes 查询以及指定的 route 信息更新的 Rest API接口

![gateway_routes_query](https://github.com/zhangwei9757/Documents/blob/master/images/gateway_core.png)

访问：  http://localhost:8080/actuator/gateway/routes 



## 5. Spring Cloud Gateway 的路由断言

> 通过上面的二种方式案例，演示了路由转发功能，下在来介绍一下其它常用的断言工厂
>
> After 路由断言工厂
>
> Before 路由断言工厂
>
> Between 路由断言工厂
>
> Cookie 路由断言工厂
>
> Header 路由断言工厂
>
> Host 路由断言工厂
>
> Method 路由断言工厂
>
> Query 路由断言工厂
>
> RemoteAddr 路由断言工厂
>
> 演示示例如下，具体效果就不一一截图，大家可以启动项目复制如下内容，测试效果 !!!

```java
@SpringBootApplication
@RestController
public class SCGatewayApplication {

    /**
	 * 测试Cookies路由断言工厂
	 * @param request
	 * @param response
	 * @return
	 */
	@GetMapping("/test/cookie")
	public String testGateway(HttpServletRequest request, HttpServletResponse response){
		Cookie[] cookies = request.getCookies();
		if (cookies != null) {
			for (Cookie cookie : cookies) {
				System.out.println(cookie.getName() + ":" + cookie.getValue());
			}
		}
		return "Spring Cloud Gateway,Hello world!";
	}


	/**
	 * 测试Head路由断言工厂
	 * @param request
	 * @param response
	 * @return
	 */
	@GetMapping("/test/head")
	public String testGatewayHead(HttpServletRequest request, HttpServletResponse response){
		String head = request.getHeader("X-Request-Id");
		return "return head info:"+head;
	}
    
    /**
	 * --------------------------- 以下为路由断言工厂演示 ---------------------------
	 */
	@Bean
	public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
        // After 路由断言工厂
		// 生成比当前时间早一个小时的UTC时间
		ZonedDateTime minusTime = LocalDateTime.now().minusHours(1).atZone(ZoneId.systemDefault());
		return builder.routes()
				.route("after_route", r -> r.after(minusTime)
						.uri("http://baidu.com"))
				.build();
	}
    
    @Bean
	public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
        // Before 路由断言工厂
		// 生成比当前时间晚一个小时的UTC时间
		ZonedDateTime datetime = LocalDateTime.now().plusDays(1).atZone(ZoneId.systemDefault());
		return builder.routes()
				.route("before_route", r -> r.before(datetime)
						.uri("http://baidu.com"))

				.build();
	}
    
    @Bean
	public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
		// Between 路由断言工厂	
		ZonedDateTime datetime1 = LocalDateTime.now().minusDays(1).atZone(ZoneId.systemDefault());
		ZonedDateTime datetime2 = LocalDateTime.now().plusDays(1).atZone(ZoneId.systemDefault());
		return builder.routes()
				.route("between_route", r -> r.between(datetime1,datetime2)
						.uri("http://baidu.com"))

				.build();
	}
    
    @Bean
	public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
        // Cookie 路由断言工厂
		return builder.routes()
				.route("cookie_route", r -> r.cookie("testCookie", "zhangwei")
						.uri("http://localhost:8080/test/cookie"))
				.build();
	}

    @Bean
	public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
        // Header 路由断言工厂
		return builder.routes()
				.route("header_route", r -> r.header("X-Request-Id", "zhangwei")
						.uri("http://localhost:8080/test/head"))
				.build();
	}
    
    @Bean
	public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
        // Host 路由断言工厂
		return builder.routes()
				.route("host_route", r -> r.host("**.baidu.com:8080")
						.uri("http://jd.com"))
				.build();
	}
    
    @Bean
	public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
        // Method 路由断言工厂
		return builder.routes()
				.route("method_route", r -> r.method("GET")
						.uri("http://jd.com"))
				.build();
	}
    
    @Bean
	public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
        // Query 路由断言工厂
		return builder.routes()
				.route("query_route", r -> r.query("foo","baz")
						.uri("http://baidu.com"))
				.build();
	}
    
    @Bean
	public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
        // RemoteAddr 路由断言工厂
		return builder.routes()
				.route("remoteaddr_route", r -> r.remoteAddr("127.0.0.1")
						.uri("http://baidu.com"))
				.route("zhangwei", r -> r.path("/zhangwei/**").
						uri("http://baidu.com"))
				.build();
	}
    
	public static void main(String[] args) {
		SpringApplication.run(SCGatewayApplication.class, args);
	}
}
```



## 6. Spring Cloud Gateway 的内置  Filter

这里简单将Spring Cloud Gateway内置的所有过滤器工厂整理成了一张表格，虽然不是很详细，但能作为速览使用。如下：

| 过滤器工厂                  | 作用                                                         | 参数                                                         |
| :-------------------------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| AddRequestHeader            | 为原始请求添加Header                                         | Header的名称及值                                             |
| AddRequestParameter         | 为原始请求添加请求参数                                       | 参数名称及值                                                 |
| AddResponseHeader           | 为原始响应添加Header                                         | Header的名称及值                                             |
| DedupeResponseHeader        | 剔除响应头中重复的值                                         | 需要去重的Header名称及去重策略                               |
| Hystrix                     | 为路由引入Hystrix的断路器保护                                | `HystrixCommand`的名称                                       |
| FallbackHeaders             | 为fallbackUri的请求头中添加具体的异常信息                    | Header的名称                                                 |
| PrefixPath                  | 为原始请求路径添加前缀                                       | 前缀路径                                                     |
| PreserveHostHeader          | 为请求添加一个preserveHostHeader=true的属性，路由过滤器会检查该属性以决定是否要发送原始的Host | 无                                                           |
| RequestRateLimiter          | 用于对请求限流，限流算法为令牌桶                             | keyResolver、rateLimiter、statusCode、denyEmptyKey、emptyKeyStatus |
| RedirectTo                  | 将原始请求重定向到指定的URL                                  | http状态码及重定向的url                                      |
| RemoveHopByHopHeadersFilter | 为原始请求删除IETF组织规定的一系列Header                     | 默认就会启用，可以通过配置指定仅删除哪些Header               |
| RemoveRequestHeader         | 为原始请求删除某个Header                                     | Header名称                                                   |
| RemoveResponseHeader        | 为原始响应删除某个Header                                     | Header名称                                                   |
| RewritePath                 | 重写原始的请求路径                                           | 原始路径正则表达式以及重写后路径的正则表达式                 |
| RewriteResponseHeader       | 重写原始响应中的某个Header                                   | Header名称，值的正则表达式，重写后的值                       |
| SaveSession                 | 在转发请求之前，强制执行`WebSession::save`操作               | 无                                                           |
| SecureHeaders               | 为原始响应添加一系列起安全作用的响应头                       | 无，支持修改这些安全响应头的值                               |
| SetPath                     | 修改原始的请求路径                                           | 修改后的路径                                                 |
| SetResponseHeader           | 修改原始响应中某个Header的值                                 | Header名称，修改后的值                                       |
| SetStatus                   | 修改原始响应的状态码                                         | HTTP 状态码，可以是数字，也可以是字符串                      |
| StripPrefix                 | 用于截断原始请求的路径                                       | 使用数字表示要截断的路径的数量                               |
| Retry                       | 针对不同的响应进行重试                                       | retries、statuses、methods、series                           |
| RequestSize                 | 设置允许接收最大请求包的大小。如果请求包大小超过设置的值，则返回 `413 Payload Too Large` | 请求包大小，单位为字节，默认值为5M                           |
| ModifyRequestBody           | 在转发请求之前修改原始请求体内容                             | 修改后的请求体内容                                           |
| ModifyResponseBody          | 修改原始响应体的内容                                         | 修改后的响应体内容                                           |
| Default                     | 为所有路由添加过滤器                                         | 过滤器工厂名称及值                                           |

**Tips：**每个过滤器工厂都对应一个实现类，并且这些类的名称必须以`GatewayFilterFactory`结尾，这是Spring Cloud Gateway的一个约定，例如`AddRequestHeader`对应的实现类为`AddRequestHeaderGatewayFilterFactory`。

在此仅为大家贴上一个作为引导，如下：

```java
public class AddRequestHeaderGatewayFilterFactory extends AbstractNameValueGatewayFilterFactory {
    public AddRequestHeaderGatewayFilterFactory() {
    }
	
    public GatewayFilter apply(NameValueConfig config) {
        return (exchange, chain) -> {
            ServerHttpRequest request = exchange.getRequest().mutate().header(config.getName(), config.getValue()).build();
            return chain.filter(exchange.mutate().request(request).build());
        };
    }
}
```



对源码感兴趣的盆友就可以按照这个规律拼接出具体的类名，以此查找这些内置过滤器工厂的实现代码



##  个人总结

> 本文旨在引导大家了解微服务 新版本 gateway 入门和一些概念，及工作原理(上文有图)，虽然不尽详细，
>
> 但是方方面面已经全部提到，更加深入的使用，如：动态配置路由，限流，大家可以在基础掌握以后，查阅
>
> 更多实现方案的资料深入的学习，万变不离其宗，只要掌握了网关的核心原理和实现方式，任何功能都可以
>
> 自由实现



## 持续更新中 ..... 