# Spring WebSocket



### 1. 概述

> 相比 HTTP 协议来说，WebSocket 协议对大多数后端开发者是比较陌生的。相比来说，WebSocket 协议**重点**是提供了服务端主动向客户端发送数据的能力，这样我们就可以完成**实时性**较高的需求。例如说，聊天 IM 即使通讯功能、消息订阅服务、网页游戏等等。
>
> 同时，因为 WebSocket 使用 TCP 通信，可以避免重复创建连接，提升通信质量和效率。
>
> 这里有个一个误区，WebSocket 相比普通的 Socket 来说，仅仅是借助 HTTP 协议完成握手，创建连接。后续的所有通信，都和 HTTP 协议无关。
>
> 官方网址: http://www.websocket.org/
>
> http 长连接协议参考: http://www.websocket.org/aboutwebsocket.html
>
> git仓库 websocket starter 源码地址: https://github.com/zhangwei9757/springcloud-zhangwei/tree/master/microservice-websocket-spring-boot-starter
>
> git仓库源码地址: https://github.com/zhangwei9757/springcloud-zhangwei/tree/master/websocket-zhangwei
>
> **说明**:  starter 包是基于原生 spring websocket 封装了一套自定义协议，简化了前后端开发的复杂程序，以及前端参数传递和身份认证， 便于开发， 维护和升级。。。



### 2. 实现方案

>方案一 : Spring WebSocket
>
>方案二 : Tomcat WebSocket
>
>方案三 : Netty WebSocket
>
>本文主要演示基于 spring 实现 webSocket 服务器



### 3.  基于 spring websocket 实现服务器



#### 3.1 创建 springboot 项目并导包

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-websocket</artifactId>
</dependency>
```



#### 3.2  TextWebSocketHandler  处理器重写，实现自定义处理功能

> 实现 WebSocketHandler 接口即可，下文有讲解把此处理器注入到websocket

```java
public class WebSocketHandler extends TextWebSocketHandler {

    @Resource
    private WebSocketServer server;

    @Override
    public void afterConnectionEstablished(WebSocketSession session) throws Exception {
        log.info(">>> incoming connection thread({}) ::: {}", Thread.currentThread().getName(), Thread.currentThread().getId());
        server.onAddSession(session);
    }

    @Override
    public void afterConnectionClosed(WebSocketSession session, CloseStatus status) throws Exception {
        log.info(">>> session closed: {}, 链接关闭,状态: {}", session.getId(), status);
        server.onDelSession(session);
    }

    @Override
    protected void handleTextMessage(WebSocketSession session, TextMessage message) throws Exception {
        try {
            String payload = message.getPayload();
            log.info(">>> from session ({}), receive text message: {}", session.getId(), message);

            int pos = payload.indexOf("|");
            if (pos == -1 || pos == payload.length() - 1) {
                return;
            }

            String type = payload.substring(0, pos);
            String body = payload.substring(pos + 1);

            server.onMessage(session, type, body);
        } catch (Exception ex) {
            log.error(">>> handleTextMessage error:", ex);
            server.onError(session, ex.getMessage());
        }
    }

    @Override
    protected void handleBinaryMessage(WebSocketSession session, BinaryMessage message) {
        try {
            ByteBuffer buffer = message.getPayload();
            log.info(">>> from session ({}), receive binary message: {}", session.getId(), message);
            server.onMessage(session, buffer);
        } catch (Exception e) {
            log.error(">>> handleBinaryMessage error:", e);
        }
    }

    @Override
    protected void handlePongMessage(WebSocketSession session, PongMessage message) throws Exception {
        log.info(">>> handlerPongMessage...");
    }

    @Override
    public void handleTransportError(WebSocketSession session, Throwable exception) throws Exception {
        log.info(">>> handleTransportError: {}", exception.getMessage());
        server.onError(session, exception.getMessage());
    }
}
```





#### 3.4  HttpSessionHandshakeInterceptor 重写拦截器，获取前端传输的参数，比如：token

```java
@Slf4j
public class WebSocketHandshakeInterceptor extends HttpSessionHandshakeInterceptor {

    @Override
    public boolean beforeHandshake(ServerHttpRequest request, ServerHttpResponse response,
                                   WebSocketHandler wsHandler, Map<String, Object> attributes) throws Exception {
        // 获得 accessToken
        if (request instanceof ServletServerHttpRequest) {
            ServletServerHttpRequest serverRequest = (ServletServerHttpRequest) request;
            String accessToken = serverRequest.getServletRequest().getParameter("accessToken");

            if (Objects.isNull(accessToken)) {
                log.error(">>> 客户端提交的数据中没有accessToken, 无法查找用户信息...");
                return false;
            }

            Map<String, String[]> paramters = serverRequest.getServletRequest().getParameterMap();
            Map<String, String> httpParams = paramters
                    .entrySet().stream()
                    .filter(entry -> entry.getValue().length > 0)
                    .collect(Collectors.toMap(Map.Entry::getKey, entry -> entry.getValue()[0]));
            attributes.putAll(httpParams);
        }

        // 调用父方法，继续执行逻辑
        return super.beforeHandshake(request, response, wsHandler, attributes);
    }

    @Override
    public void afterHandshake(ServerHttpRequest request, ServerHttpResponse response, WebSocketHandler wsHandler, Exception ex) {
        super.afterHandshake(request, response, wsHandler, ex);
    }
}
```



#### 3.5 实现一个会话管理工具类，即可实现单服务器，多客户端功能， 一对多，多对多 通信功能

```java
// 此处不涉及，多线程，所以不用考虑安全问题，只需考虑性能即可
protected HashMap<String, WebSocketUser> usersSessionMap = new HashMap<>();
// 基于客户端上，下线，处理见源码
```



#### 3.6 配置前面的拦截器和处理器

> 实现 WebSocketConfigurer 接口, 添加 @EnableWebSocket注解

```java 
@Slf4j
@Configuration
@EnableWebSocket
public class WebSocketConfiguration implements WebSocketConfigurer {

    @Resource
    private WebsocketEndpointProperties websocketEndpointProperties;

    private final Object monitor = new Object();

    private void createWebsocketEndpointProperties() {
        synchronized (monitor) {
            if (null == websocketEndpointProperties) {
                websocketEndpointProperties = new WebsocketEndpointProperties();
                log.info(">>> WebsocketEndpointProperties Successfully initialized: {}", websocketEndpointProperties);
            }
        }
    }

    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry webSocketHandlerRegistry) {
        webSocketHandlerRegistry.addHandler(webSocketHandler(),
                "socket")
                .addInterceptors(webSocketHandshakeInterceptor())
                .setAllowedOrigins("*")
                .withSockJS();
        log.info(">>> WebSocketConfiguration Successfully register registerWebSocketHandlers...");
    }

    @Bean
    public WebSocketHandler webSocketHandler() {
        return new WebSocketHandler();
    }

    @Bean
    public WebSocketHandshakeInterceptor webSocketHandshakeInterceptor() {
        return new WebSocketHandshakeInterceptor();
    }
}
```



### 4.  websocket 错误码列表

| 状态码    | 名称                 | 描述                                                         |                        |
| --------- | -------------------- | ------------------------------------------------------------ | ---------------------- |
| 0–999     | -                    | 保留段, 未使用。                                             |                        |
| 1000      | CLOSE_NORMAL         | 正常关闭; 无论为何目的而创建, 该链接都已成功完成任务。       |                        |
| 1001      | CLOSE_GOING_AWAY     | 终端离开, 可能因为服务端错误, 也可能因为浏览器正从打开连接的页面跳转离开。 |                        |
| 1002      | CLOSE_PROTOCOL_ERROR | 由于协议错误而中断连接。                                     |                        |
| 1003      | CLOSE_UNSUPPORTED    | 由于接收到不允许的数据类型而断开连接 (如仅接收文本数据的终端接收到了二进制数据)。 |                        |
| 1004      | -                    | 保留。 其意义可能会在未来定义。                              |                        |
| 1005      | CLOSE_NO_STATUS      | 保留。 表示没有收到预期的状态码。                            |                        |
| 1006      | CLOSE_ABNORMAL       | 保留。 用于期望收到状态码时连接非正常关闭 (也就是说, 没有发送关闭帧)。 |                        |
| 1007      | Unsupported Data     | 由于收到了格式不符的数据而断开连接 (如文本消息中包含了非 UTF-8 数据)。 |                        |
| 1008      | Policy Violation     | 由于收到不符合约定的数据而断开连接。 这是一个通用状态码, 用于不适合使用 1003 和 1009 状态码的场景。 |                        |
| 1009      | CLOSE_TOO_LARGE      | 由于收到过大的数据帧而断开连接。                             |                        |
| 1010      | Missing Extension    | 客户端期望服务器商定一个或多个拓展, 但服务器没有处理, 因此客户端断开连接。 |                        |
| 1011      | Internal Error       | 客户端由于遇到没有预料的情况阻止其完成请求, 因此服务端断开连接。 |                        |
| 1012      | Service Restart      | 服务器由于重启而断开连接。 [Ref]                             |                        |
| 1013      | Try Again Later      | 服务器由于临时原因断开连接, 如服务器过载因此断开一部分客户端连接。 [Ref] |                        |
| 1014      | -                    | 由 WebSocket                                                 | 标准保留以便未来使用。 |
| 1015      | TLS Handshake        | 保留。 表示连接由于无法完成 TLS 握手而关闭 (例如无法验证服务器证书)。 |                        |
| 1016–1999 | -                    | 由 WebSocket 标准保留以便未来使用。                          |                        |
| 2000–2999 | -                    | 由 WebSocket 拓展保留使用。                                  |                        |
| 3000–3999 | -                    | 可以由库或框架使用。 不应由应用使用。 可以在 IANA 注册, 先到先得。 |                        |
| 4000–4999 | -                    | 可以由应用使用。                                             |                        |





### 5. 测试 websocket 连接

> 提示:  此处未展示具体细节，处理器里的连接，接收事件等等，可以自己简单的实现并启动服务器。
>
> 可以使用在线工具前端页面:  http://www.easyswoole.com/wstool.html



```json
1. 使用链接url:  ws://127.0.0.1:8080/socket 尝试连接

2. 返回结果如下提示: 
17:26:19 => 初始化完成
18:09:51 => 发生错误 请打开浏览器控制台查看
18:09:51 => CLOSED => 1006 CLOSE_ABNORMAL

3. 通过错误码列表以及后台服务器日志, 我们知道是没有提供身份认证的参数: accessToken
2020-10-06 18:09:51.362 ERROR [http-nio-8080-exec-1] --- c.z.w.WebSocketHandshakeInterceptor Line:31  - >>> 客户端提交的数据中没有accessToken, 无法查找用户信息...

4. 再次尝试链接: ws://127.0.0.1:8080/socket?accessToken=test
前端日志：18:14:29 => OPENED => 127.0.0.1:8080/socket?accessToken=test
后台日志: 
2020-10-06 18:14:29.308 INFO  [http-nio-8080-exec-2] --- c.z.websocket.WebSocketHandler Line:23  - >>> incoming connection thread(http-nio-8080-exec-2) ::: 83
2020-10-06 18:14:29.319 INFO  [http-nio-8080-exec-2] --- c.z.websocket.WebSocketServer2 Line:45  - -------------------- 当前在线用户列表：---------------------
2020-10-06 18:14:29.320 INFO  [http-nio-8080-exec-2] --- c.z.websocket.WebSocketServer2 Line:-1  - test
2020-10-06 18:14:29.320 INFO  [http-nio-8080-exec-2] --- c.z.websocket.WebSocketServer2 Line:47  - ---------------------------------------------------------

5. 目前我们已成功建立连接通道

6. 我们使用源码里提供的自定义协议，传输文本内容: AuthRequest|{"accessToken":"test"}
   返回结果: 
    收到消息 18:22:18
    Return|{"type":"AuthRequest","result":"2020-10-06T18:22:18.824:::test"}
7. 此时我们就简单的实现了前后端自定义协议，传输报文，同理也可以实现二进制小文本传输方案, 

8. 大文件可以使用浏览器天生的处理能力，后台使用springMVC接收大文件传输，或者使用开源的第三方，前后端自定义方案进行断点续传功能，会极大的提供服务器传输文件和适用性，以及性能和并发
```



### 6. 自定义协议运行流程

> 1.  基于 springboot 创建项目后，创建协议处理目录包:  protos
> 2.  创建java 类继续 BaseProtocol 类， 重写： onProcess 方法，或者也可以重写预处理preProcess，后处理逻辑postProcess
> 3.  我们只需要关注 protos下的处理实现类，即可与前端拥有交互能力，且无代码侵入，组件式插拔，即使后续我们不使用 starter组件，只需要迁移 protos包下的处理逻辑即可。方便项目升级，迁移或者重组。
