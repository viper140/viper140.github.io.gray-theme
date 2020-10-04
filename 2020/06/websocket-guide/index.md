---
title: 'WebSocket 入坑必看'
date: 2020-06-14T12:00:00+08:00
lastmod: 2020-06-14T12:00:00+08:00
tags: ['Java', 'WebSocket', 'Http']
categories: ['开发']
author: '王峰'
---

- 介绍 WebSocket 的原理，了解原理后，用起来更放心大胆
- 类似技术对比，搞清楚自己的业务场景是不是需要使用 WebSocket
- 使用过程中的经验分享，让你少走一些弯路

<!--more-->

## 1 WebSocket 是什么

WebSocket 是 HTML5 新增的在单个 TCP 连接上进行**全双工通讯**（不受限的双向通信）的协议，能更好的节省服务器资源和带宽，并且能够更实时地进行通讯。

> 全双工（Full Duplex）的通讯传输允许数据在两个方向上同时传输，相当于两个单工通信方式的结合。发送和接收分别由两根不同的传输线传送，通信双方既是发送器也是接收器。

Websocket 使用和 HTTP 相同的 TCP 端口，可以绕过大多数防火墙的限制。默认情况下，Websocket 协议使用 80 端口；运行在 TLS 上则使用 443 端口。

```text
常见问题：

Q：WebSocket 能全双工，为何普通 HTTP 请求不行？（他们建立在 TCP 协议之上的，TCP 协议本就实现了全双工通信）
A：其实是 HTTP 的“请求－应答模式”限制了 TCP 协议本支持的全双工通信。

Q：WebSocket 和 Socket 的区别
A：Socket 不是协议，是应用层与 TCP/IP 通信的中间软件抽象层，是一组接口。而 WebSocket 是应用层协议。

Q：WebSocket 长连接和 HTTP 长连接的区别
A：HTTP/1.1 默认开启了长连接（Connection:keep-alive），本质是 TCP 长连接，可在一次 TCP 连接中完成多个 HTTP 请求。
WebSocket 的长连接是真正的全双工，TCP 链路建立后，双方可以互发消息，无需再设置请求头，且双方都需要维持住这个连接。

关于 HTTP 长连接再多说几句，打开浏览器控制台 network，每个请求都会有个 Connection ID，这表示 TCP 连接的 id，会发现可能多个 HTTP 请求的 Connection ID 是一样的，这代表他们共用一个 TCP 连接。
另外 chrome 允许一个域名有 6 个 TCP 连接并发，意味着同时发出的请求超过这个数字，只能排队了
```

## 2 为什么要用 WebSocket

### 2.1 需求描述、应用场景

- 需求：服务端数据更新，需要通知到客户端。

- 应用场景：聊天软件、订阅、游戏、协同工作（比如文本编辑）、直播、股票基金、基于位置的应用等。

### 2.2 常用解决方案对比

WebSocket 能解决上述需求，除此之外，常用的解决方案还有：轮询、长轮询。另外 html5 还提供了 Server-Sent Event。

- 轮询：客户端定时向服务端发送 http 请求，服务端收到请求后立即返回响应信息并关闭连接；
- 长轮询：为了解决轮询无效请求过多的问题，长轮询进行了优化，服务端收到请求后先阻塞，必要时再返回数据并关闭连接，客户端处理完响应信息后才再向服务端发送新的请求；
- Server-Sent Event：html5 提供的，借用了长轮询的思想，但不再每个连接只收发一个消息，将文本数据换成流以实现重复在一个连接上收发消息；

| 常用方案          | 通讯方式   | 触发方式 | 缺点                                                          | 优点                             |
| ----------------- | ---------- | -------- | ------------------------------------------------------------- | -------------------------------- |
| 轮询              | http       | 轮询     | 服务端不能主动推送；消息不及时；浪费带宽                      | 实现容易                         |
| 长轮询            | http       | 轮询     | 服务端仍不能主动推送；占用 web 连接                           | 实现较容易                       |
| Server-Sent Event | http       | 事件     | 兼容性问题（不支持 ie）； 占用 web 连接；只能服务端向客户端推 | 实现较容易；自动重连             |
| WebSocket         | tcp 长连接 | 事件     | 开发成本高                                                    | 全双工；安全性高；节约带宽和资源 |

- Server-Sent Event 一个用于微信支付的案例：[https://www.jianshu.com/p/9a0e802b2297](https://www.jianshu.com/p/9a0e802b2297)

```text
SSE suffers from a limitation to the maximum number of open connections, which can be specially painful when opening various tabs as the limit is per browser and set to a very low number (6). The issue has been marked as "Won't fix" in Chrome and Firefox
```

- 长轮询和 SSE 会占用浏览器有限的连接数（chrome 有 6 个），看起来很致命啊

> 另外 HTTP/2 提供了服务器推送(Server Push)的功能，千万别和上面几个东西搞混了，完全不是一回事。服务器指的是 web 服务器，推送的对象是浏览器要加载的资源，是用于提升首屏加载速度的技术，需要在 web 服务器（比如 nginx）中开启相关配置。可以参考这篇：[https://www.cnblogs.com/wetest/p/8040202.html](https://www.cnblogs.com/wetest/p/8040202.html)

## 3 WebSocket 连接建立过程

`WebSocket并不是全新的协议，而是利用了HTTP协议来建立连接。我们来看看WebSocket连接是如何创建的。`

### 3.1 浏览器发起一个 http 请求建立连接

请求地址以`ws://`开头，请求头`Upgrade: websocket`和`Connection: Upgrade`表示这个连接将要被转换为 WebSocket 连接。

- 1.1 建立 TCP 连接

- 1.2 浏览器发送 HTTP 请求，并携带协议升级的头信息，进行协议升级前的握手

### 3.2 服务器响应请求

响应头`HTTP/1.1 101 Switching Protocols`和`Upgrade: websocket`表示本次连接的 HTTP 协议即将被更改（代码 101），改为指定的 WebSocket 协议。

- 2.1 响应 HTTP 握手，返回 code 101

- 2.2 双方可以通过这个连接自由的传信息，连接会持续存在，server 和 client 都可单方面断开连接

## 4 使用需知 & 实用指南

### 4.1 正确使用 ws 和 wss

- WebSocket 的协议标识符是`ws`，如果在 TLS 协议上，标识符是`wss`，类似于 https

- https 下必须使用 wss 作为安全链接

> TLS 之上的 Websocket：首先，浏览器用 wss://xxx 创建 WebSocket 连接时，会先通过 HTTPS 创建安全的连接，然后，该 HTTPS 连接升级为 WebSocket 连接，底层通信走的仍然是安全的 SSL/TLS 协议。

### 4.2 使用 Nginx 代理 WebSocket 请求

- Nginx 从 1.3 开始就支持 WebSocket 了，并且可以为 WebSocket 应用程序做反向代理和负载均衡。官方文档：[http://nginx.org/en/docs/http/websocket.html](http://nginx.org/en/docs/http/websocket.html)

- 当客户端发过来一个协议升级的 http 请求时，Nginx 默认是不知道的，需要配置`proxy_set_header Upgrade $http_upgrade`和`proxy_set_header Connection "Upgrade"`，
  配置后，当 Nginx 代理服务器拦截到客户端发来的 Upgrade 请求时，会使用 101（交换协议）返回响应，在客户端和代理服务器、后端服务器之间建立隧道来支持 WebSocket。

- 配置示例：

```conf
server {
    listen       80;
    server_name dev-staff-api-gateway.teyixing.com;
    rewrite ^(.*)$ https://${server_name}$1 permanent;
}
server {
    server_name dev-staff-api-gateway.teyixing.com;
    listen 443 http2 ssl;

    ssl_certificate conf.d/cert/teyixing.com.pem;
    ssl_certificate_key conf.d/cert/teyixing.com.key;
    ssl_session_timeout 5m;
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_prefer_server_ciphers on;

    # 用于 WebSocket
    location /v1/webSocket {
        proxy_pass http://dev-staff-api-gateway/v1/webSocket; # http://call-center/v1/webSocket;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "Upgrade";
    }

    # 拦截普通 http 请求
    location / {
        proxy_pass http://dev-staff-api-gateway;
    }
}
```

### 4.3 如何解决 nginx 掐断 WebSocket 连接的问题

#### 4.3.1 问题简述

有时候会发现 WebSocket 连接莫名其妙断了，后端日志发现有如下报错：

```log
com.tehang.callcenter.application.websocket.WebSocketConnection.onError
java.io.EOFException: null
	at org.apache.tomcat.util.net.NioEndpoint$NioSocketWrapper.fillReadBuffer(NioEndpoint.java:1208)
	at org.apache.tomcat.util.net.NioEndpoint$NioSocketWrapper.read(NioEndpoint.java:1142)
	at org.apache.tomcat.websocket.server.WsFrameServer.onDataAvailable(WsFrameServer.java:72)
	at org.apache.tomcat.websocket.server.WsFrameServer.doOnDataAvailable(WsFrameServer.java:171)
	at org.apache.tomcat.websocket.server.WsFrameServer.notifyDataAvailable(WsFrameServer.java:151)
	at org.apache.tomcat.websocket.server.WsHttpUpgradeHandler.upgradeDispatch(WsHttpUpgradeHandler.java:148)
	at org.apache.coyote.http11.upgrade.UpgradeProcessorInternal.dispatch(UpgradeProcessorInternal.java:54)
	at org.apache.coyote.AbstractProcessorLight.process(AbstractProcessorLight.java:53)
	at org.apache.coyote.AbstractProtocol$ConnectionHandler.process(AbstractProtocol.java:834)
	at org.apache.tomcat.util.net.NioEndpoint$SocketProcessor.doRun(NioEndpoint.java:1417)
	at org.apache.tomcat.util.net.SocketProcessorBase.run(SocketProcessorBase.java:49)
	at java.base/java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1128)
	at java.base/java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:628)
	at org.apache.tomcat.util.threads.TaskThread$WrappingRunnable.run(TaskThread.java:61)
	at java.base/java.lang.Thread.run(Thread.java:834)
```

#### 4.3.2 原因

- nginx 配置项 proxy_read_timeout 的默认值为 60s，表示等待服务器响应的时间。也就是说，当 WebSocket 使用 nginx 转发时，如 60s 内没有通讯，nginx 便会掐断连接。

#### 4.3.3 解决方案

- nginx proxy_read_timeout 设置为不超时
- 前端发起心跳检测
- 前端在 WebSocket 生命周期方法 onError 中调用 reconnect
