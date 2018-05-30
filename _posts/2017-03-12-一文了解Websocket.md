---
layout: post
title: "一文了解Websocket"
author: "Gerald"
---

#### 简介

WebSocket 协议在2008年诞生，2011年成为国际标准。所有浏览器都已经支持了。WebSocket使得客户端和服务器之间的数据交换变得更加简单，允许服务端主动向客户端推送数据。在WebSocket API中，浏览器和服务器只需要完成一次握手，两者之间就直接可以创建持久性的连接，并进行双向数据传输。

<!--more-->

#### 主要特点

它的最大特点就是，服务器可以主动向客户端推送信息，客户端也可以主动向服务器发送信息，是真正的双向平等对话，属于服务器推送技术的一种。其他特点还包括：

- 建立在 TCP 协议之上，服务器端的实现比较容易。

- 与 HTTP 协议有着良好的兼容性。默认端口也是80和443，并且握手阶段采用 HTTP 协议，因此握手时不容易屏蔽，能通过各种 HTTP 代理服务器。

- 数据格式比较轻量，性能开销小，通信高效。

- 可以发送文本，也可以发送二进制数据。

- 没有同源限制，客户端可以与任意服务器通信。

- 协议标识符是ws（如果加密，则为wss），服务器网址就是 URL。

#### 握手协议

WebSocket 是独立的、创建在 TCP 上的协议。

Websocket 通过 HTTP/1.1 协议的101状态码进行握手。

为了创建Websocket连接，需要通过浏览器发出请求，之后服务器进行回应，这个过程通常称为“握手”（handshaking）。

一个典型的Websocket握手请求如下：

**客户端请求**

```
GET / HTTP/1.1
Upgrade: websocket
Connection: Upgrade
Host: example.com
Origin: http://example.com
Sec-WebSocket-Key: sN9cRrP/n9NdMgdcy2VJFQ==
Sec-WebSocket-Version: 13
```

**服务器回应**

```
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: fFBooB7FAkLlXgRSz0BT3v4hq5s=
Sec-WebSocket-Location: ws://example.com/
```

**字段解析**

- Connection必须设置Upgrade，表示客户端希望连接升级。

- Upgrade字段必须设置Websocket，表示希望升级到Websocket协议。

- Sec-WebSocket-Key是随机的字符串，服务器端会用这些数据来构造出一个SHA-1的信息摘要。把“Sec-WebSocket-Key”加上一个特殊字符串“258EAFA5--E914-47DA-95CA-C5AB0DC85B11”，然后计算SHA-1摘要，之后进行BASE-64编码，将结果做为“Sec-WebSocket-Accept”头的值，返回给客户端。如此操作，可以尽量避免普通HTTP请求被误认为Websocket协议。

- Sec-WebSocket-Version 表示支持的Websocket版本。RFC6455要求使用的版本是13，之前草案的版本均应当弃用。

- Origin字段是可选的，通常用来表示在浏览器中发起此Websocket连接所在的页面，类似于Referer。但是，与Referer不同的是，Origin只包含了协议和主机名称。

- 其他一些定义在HTTP协议中的字段，如Cookie等，也可以在Websocket中使用。

#### 应用场景

- 即时聊天通信

- 多玩家游戏

- 在线协同编辑/编辑

- 实时数据流的拉取与推送

- 体育/游戏实况

- 实时地图位置
