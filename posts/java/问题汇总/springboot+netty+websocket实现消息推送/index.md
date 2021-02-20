# Springboot+netty+websocket实现消息推送


> 参考：  
> https://juejin.cn/post/6844904110576107534  


实现思路：
1. 前端使用webSocket与服务端创建连接的时候，将用户ID传给服务端
2. 服务端将用户ID与channel关联起来存储，同时将channel放入到channel组中
3. 如果需要给所有用户发送消息，直接执行channel组的writeAndFlush()方法
4. 如果需要给指定用户发送消息，根据用户ID查询到对应的channel,然后执行writeAndFlush()方法
5. 前端获取到服务端推送的消息之后，将消息内容展示到文本域中

## 引入 netty 依赖

pom.xml 文件主要内容如下

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>

    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
    </dependency>

    <dependency>
        <groupId>io.netty</groupId>
        <artifactId>netty-all</artifactId>
    </dependency>
</dependencies>
```

## 定义 netty 服务端缓存

存储客户端的 id 等信息

```java
package org.msdemt.simple.config;

import io.netty.channel.Channel;
import io.netty.channel.group.ChannelGroup;
import io.netty.channel.group.DefaultChannelGroup;
import io.netty.util.concurrent.GlobalEventExecutor;

import java.util.concurrent.ConcurrentHashMap;

public class NettyChannelCache {

    /**
     * 定义一个channel组，管理所有的channel
     * GlobalEventExecutor.INSTANCE 是全局的事件执行器，是一个单例
     */
    private static ChannelGroup channelGroup = new DefaultChannelGroup(GlobalEventExecutor.INSTANCE);

    /**
     * 存放用户与Chanel的对应信息，用于给指定用户发送消息
     */
    private static ConcurrentHashMap<String, Channel> userChannelMap = new ConcurrentHashMap<>();

    /**
     * 获取channel组
     *
     * @return
     */
    public static ChannelGroup getChannelGroup() {
        return channelGroup;
    }

    /**
     * 获取用户channel map
     *
     * @return
     */
    public static ConcurrentHashMap<String, Channel> getUserChannelMap() {
        return userChannelMap;
    }
}
```

## 定义 netty server

```java
package org.msdemt.simple.config;

import io.netty.bootstrap.ServerBootstrap;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.EventLoopGroup;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioServerSocketChannel;
import io.netty.handler.codec.http.HttpObjectAggregator;
import io.netty.handler.codec.http.HttpServerCodec;
import io.netty.handler.codec.http.websocketx.WebSocketServerProtocolHandler;
import io.netty.handler.codec.serialization.ObjectEncoder;
import io.netty.handler.stream.ChunkedWriteHandler;
import io.netty.handler.timeout.IdleStateHandler;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

import javax.annotation.PostConstruct;
import javax.annotation.PreDestroy;
import java.net.InetSocketAddress;

@Slf4j
@Component
public class NettyServer {

    /**
     * webSocket协议名
     */
    private static final String WEBSOCKET_PROTOCOL = "WebSocket";

    /**
     * 端口号
     */
    @Value("${webSocket.netty.port:58080}")
    private int port;

    /**
     * webSocket路径
     */
    @Value("${webSocket.netty.path:/webSocket}")
    private String webSocketPath;

    /**
     * 在Netty心跳检测中配置 - 读空闲超时时间设置
     */
    @Value("${webSocket.netty.readerIdleTime:10}")
    private Integer readerIdleTime;

    /**
     * 在Netty心跳检测中配置 - 写空闲超时时间设置
     */
    @Value("${webSocket.netty.writerIdleTime:0}")
    private Integer writerIdleTime;

    /**
     * 在Netty心跳检测中配置 - 读写空闲超时时间设置
     */
    @Value("${webSocket.netty.allIdleTime:0}")
    private Integer allIdleTime;

    private WebSocketHandler webSocketHandler;

    @Autowired
    public void setWebSocketHandler(WebSocketHandler webSocketHandler) {
        this.webSocketHandler = webSocketHandler;
    }

    //bossGroup辅助客户端的tcp连接请求
    private EventLoopGroup bossGroup;
    //workGroup负责与客户端之前的读写操作
    private EventLoopGroup workGroup;

    /**
     * 启动
     *
     * @throws InterruptedException
     */
    private void start() throws InterruptedException {
        bossGroup = new NioEventLoopGroup();
        workGroup = new NioEventLoopGroup();
        ServerBootstrap bootstrap = new ServerBootstrap();
        bootstrap.group(bossGroup, workGroup);
        // 设置NIO类型的channel
        bootstrap.channel(NioServerSocketChannel.class);
        // 设置监听端口
        bootstrap.localAddress(new InetSocketAddress(port));
        // 连接到达时会创建一个通道
        bootstrap.childHandler(new ChannelInitializer<SocketChannel>() {

            @Override
            protected void initChannel(SocketChannel sh) throws Exception {
                // 流水线管理通道中的处理程序（Handler），用来处理业务
                // webSocket协议本身是基于http协议的，所以这边也要使用http编解码器
                sh.pipeline().addLast(new HttpServerCodec());
                sh.pipeline().addLast(new ObjectEncoder());
                // 以块的方式来写的处理器
                sh.pipeline().addLast(new ChunkedWriteHandler());
                /*
                    说明：
                    1. http数据在传输过程中是分段的，HttpObjectAggregator可以将多个段聚合
                    2. 这就是为什么，当浏览器发送大量数据时，就会发送多次http请求
                 */
                sh.pipeline().addLast(new HttpObjectAggregator(8192));
                /*
                    说明：
                    1. 对应webSocket，它的数据是以帧（frame）的形式传递
                    2. 浏览器请求时 ws://localhost:58080/xxx 表示请求的uri
                    3. 核心功能是将http协议升级为ws协议，保持长连接
                */
                sh.pipeline().addLast(new WebSocketServerProtocolHandler(webSocketPath, WEBSOCKET_PROTOCOL,
                        true, 65536 * 10));
                //针对客户端，若10s内无读事件则触发心跳处理方法HeartBeatHandler#userEventTriggered
                sh.pipeline().addLast(new IdleStateHandler(readerIdleTime, writerIdleTime, allIdleTime));
                // 自定义的handler，处理业务逻辑
                sh.pipeline().addLast(webSocketHandler);
            }
        });
        // 配置完成，开始绑定server，通过调用sync同步方法阻塞直到绑定成功
        ChannelFuture channelFuture = bootstrap.bind().sync();
        log.info("Server started and listen on:{}", channelFuture.channel().localAddress());
        // 对关闭通道进行监听
        channelFuture.channel().closeFuture().sync();
    }

    /**
     * 释放资源
     *
     * @throws InterruptedException
     */
    @PreDestroy
    public void destroy() throws InterruptedException {
        if (bossGroup != null) {
            bossGroup.shutdownGracefully().sync();
        }
        if (workGroup != null) {
            workGroup.shutdownGracefully().sync();
        }
    }

    /**
     * 初始化(新线程开启)
     * 需要开启一个新的线程来执行netty server，要不然会阻塞主线程
     */
    @PostConstruct()
    public void init() {
        //需要开启一个新的线程来执行netty server 服务器
        new Thread(() -> {
            try {
                start();
            } catch (InterruptedException e) {
                log.error("netty server start failed, error: " + e.getMessage());
                e.printStackTrace();
            }
        }).start();
    }
}
```

## 定义 websocket 处理器

```java
package org.msdemt.simple.config;

import com.fasterxml.jackson.databind.ObjectMapper;
import io.netty.channel.*;
import io.netty.handler.codec.http.websocketx.TextWebSocketFrame;
import io.netty.handler.timeout.IdleState;
import io.netty.handler.timeout.IdleStateEvent;
import io.netty.util.AttributeKey;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;

import java.util.Map;

/**
 * TextWebSocketFrame类型， 表示一个文本帧
 */
@Slf4j
@Component
@ChannelHandler.Sharable
public class WebSocketHandler extends SimpleChannelInboundHandler<TextWebSocketFrame> {

    /**
     * 一旦连接,第一个被执行
     *
     * @param ctx
     * @throws Exception
     */
    @Override
    public void handlerAdded(ChannelHandlerContext ctx) throws Exception {
        log.info("handlerAdded 被调用" + ctx.channel().id().asLongText());
        // 添加到channelGroup 通道组
        NettyChannelCache.getChannelGroup().add(ctx.channel());
    }

    /**
     * 读取数据
     */
    @Override
    protected void channelRead0(ChannelHandlerContext ctx, TextWebSocketFrame msg) throws Exception {
        // 获取用户ID,关联channel
        ObjectMapper objectMapper = new ObjectMapper();
        Map<String, String> map = objectMapper.readValue(msg.text(), Map.class);
        String uid = map.get("uid");

        // 当用户ID已存入通道内,则不进行写入,只有第一次建立连接时才会存入,其他情况发送uid则为心跳需求
        if (!NettyChannelCache.getUserChannelMap().containsKey(uid)) {
            log.info("客户端连接：{}", msg.text());
            NettyChannelCache.getUserChannelMap().put(uid, ctx.channel());
            // 将用户ID作为自定义属性加入到channel中，方便随时channel中获取用户ID
            AttributeKey<String> key = AttributeKey.valueOf("userId");
            ctx.channel().attr(key).setIfAbsent(uid);
            // 回复消息
            ctx.channel().writeAndFlush(new TextWebSocketFrame("服务器连接成功！"));
        } else {
            // 前端定时请求,保持心跳连接,避免服务端误删通道
            ctx.channel().writeAndFlush(new TextWebSocketFrame("keep alive success！"));
        }
    }

    /**
     * 移除通道及关联用户
     *
     * @param ctx
     * @throws Exception
     */
    @Override
    public void handlerRemoved(ChannelHandlerContext ctx) throws Exception {
        log.info("handlerRemoved 被调用" + ctx.channel().id().asLongText());
        // 删除通道
        NettyChannelCache.getChannelGroup().remove(ctx.channel());
        removeUserId(ctx);
    }

    /**
     * 异常处理
     *
     * @param ctx
     * @param cause
     * @throws Exception
     */
    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        log.info("异常：{}", cause.getMessage());
        // 删除通道
        NettyChannelCache.getChannelGroup().remove(ctx.channel());
        removeUserId(ctx);
        ctx.close();
    }

    /**
     * 心跳检测相关方法
     *
     * @param ctx
     * @param evt
     * @throws Exception
     */
    @Override
    public void userEventTriggered(final ChannelHandlerContext ctx, Object evt) throws Exception {
        log.info("heart beat...");
        if (evt instanceof IdleStateEvent) {
            IdleStateEvent event = (IdleStateEvent) evt;
            if (event.state() == IdleState.ALL_IDLE) {
                //清除超时会话
                ChannelFuture writeAndFlush = ctx.writeAndFlush("you will close");
                writeAndFlush.addListener(new ChannelFutureListener() {

                    @Override
                    public void operationComplete(ChannelFuture future) throws Exception {
                        ctx.channel().close();
                    }
                });
            }
        } else {
            super.userEventTriggered(ctx, evt);
        }
    }

    /**
     * 删除用户与channel的对应关系
     *
     * @param ctx
     */
    private void removeUserId(ChannelHandlerContext ctx) {
        AttributeKey<String> key = AttributeKey.valueOf("userId");
        String userId = ctx.channel().attr(key).get();
        NettyChannelCache.getUserChannelMap().remove(userId);
        log.info("删除用户与channel的对应关系,uid：{}", userId);
    }
}
```

## 消息推送业务接口及实现

```java
package org.msdemt.simple.service;

public interface PushService {

    /**
     * 推送给指定用户
     * @param userId 用户ID
     * @param msg 消息信息
     */
    void pushMsgToOne(String userId,String msg);

    /**
     * 推送给所有用户
     * @param msg 消息信息
     */
    void pushMsgToAll(String msg);

    /**
     * 获取当前连接数
     * @return 连接数
     */
    int getConnectCount();
}
```

```java
package org.msdemt.simple.service.impl;

import io.netty.channel.Channel;
import io.netty.handler.codec.http.websocketx.TextWebSocketFrame;
import org.msdemt.simple.config.NettyChannelCache;
import org.msdemt.simple.service.PushService;
import org.springframework.stereotype.Service;

import java.util.concurrent.ConcurrentHashMap;

@Service("pushService")
public class PushServiceImpl implements PushService {

    /**
     * 推送给指定用户
     * @param userId 用户ID
     * @param msg 消息信息
     */
    @Override
    public void pushMsgToOne(String userId, String msg){
        ConcurrentHashMap<String, Channel> userChannelMap = NettyChannelCache.getUserChannelMap();
        Channel channel = userChannelMap.get(userId);
        channel.writeAndFlush(new TextWebSocketFrame(msg));
    }

    /**
     * 推送给所有用户
     * @param msg 消息信息
     */
    @Override
    public void pushMsgToAll(String msg){
        NettyChannelCache.getChannelGroup().writeAndFlush(new TextWebSocketFrame(msg));
    }

    /**
     * 获取当前连接数
     * @return 连接数
     */
    @Override
    public int getConnectCount() {
        return NettyChannelCache.getChannelGroup().size();
    }
}
```

## controller 层提供消息发送调用接口

```java
package org.msdemt.simple.controller;

import org.msdemt.simple.service.PushService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/push")
public class PushController {

    private PushService pushService;

    @Autowired
    public PushController(PushService pushService) {
        this.pushService = pushService;
    }

    /**
     * 推送给所有用户
     * @param msg 消息信息
     */
    @PostMapping("/pushAll")
    public void pushToAll(@RequestParam("msg") String msg){
        pushService.pushMsgToAll(msg);
    }

    /**
     * 推送给指定用户
     * @param userId 用户ID
     * @param msg 消息信息
     */
    @PostMapping("/pushOne")
    public void pushMsgToOne(@RequestParam("userId") String userId,@RequestParam("msg") String msg){
        pushService.pushMsgToOne(userId,msg);
    }

    /**
     * 获取当前连接数
     */
    @GetMapping("/getConnectCount")
    public int getConnectCout(){
        return pushService.getConnectCount();
    }
}
```


## html 页面模拟客户端

`push.html` 位于 `src/main/resources/static` 目录下

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <title>消息推送</title>
</head>
<body>

<script>
    let socket;
    // 判断当前浏览器是否支持webSocket
    if(window.WebSocket){
        socket = new WebSocket("ws://localhost:58080/webSocket")
        // 相当于channel的read事件，ev 收到服务器回送的消息
        socket.onmessage = function (ev) {
            let rt = document.getElementById("responseText");
            rt.value = rt.value + "\n" + ev.data;
        }
        // 相当于连接开启
        socket.onopen = function (ev) {
            let rt = document.getElementById("responseText");
            rt.value =  "连接开启了..."
            socket.send(
                JSON.stringify({
                    // 连接成功将，用户ID传给服务端
                    uid: "123456"
                })
            );
        }
        // 相当于连接关闭
        socket.onclose = function (ev) {
            let rt = document.getElementById("responseText");
            rt.value = rt.value + "\n" + "连接关闭了...";
        }
    }else{
        alert("当前浏览器不支持webSocket")
    }
</script>

<form onsubmit="return false">
    <textarea id="responseText" style="height: 150px; width: 300px;"></textarea>
    <input type="button" value="清空内容" onclick="document.getElementById('responseText').value=''">
</form>

</body>
</html>
```

## 启动类

```java
package org.msdemt.simple;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class MessagePushApplication {
    public static void main(String[] args) {
        SpringApplication.run(MessagePushApplication.class, args);
    }
}
```

## 测试

启动后，访问 `http://localhost:8080/push.html` ，客户端会将 userId 为 123456 注册到 netty server 中，并由 netty 保证心跳。

使用 post 请求 controller 发送消息

使用 idea httpclient 进行测试，新建文件 `test.http` ，加入如下内容

```
POST http://localhost:8080/push/pushOne
Content-Type: application/x-www-form-urlencoded

msg=你好,世界&userId=123456
```

执行后，可以看到 html 页面收到了`你好，世界`的消息


