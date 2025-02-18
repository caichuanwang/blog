---
title: websocket
date: 2025-02-18 10:11:55
tags: linux
description: 怎么解决服务端主动向客户端发送消息的问题？websocket怎么使用，原理是什么？
categories:
---

# webScoket

## 前言

随着web的发展，用户对于web的要求越来越高，客户端和服务端之间的及时通行并拿到最新的数据就显得非常重要，但是，我们通用进行客户端和服务端通信的协议一般都是http或者https，但是无论http的1.0,1.1等版本，都是通信只能由客户端发起，服务端没法主动给客户端推送。那如果现在有个对实时性要求比较高的场景，比如即时通信、实况直播等情况下，我们能只能使用轮询或者长轮询的方式解决，但是如果使用webScoket的话，就可以很好解决这个问题。

## http的解决方案

### 轮询和长轮询

轮询，顾名思义，就是每隔一段时间，客户端就向服务端询问一次。而长轮询则是对轮询的改进版，客户端发送 HTTP 给服务器之后，看有没有新消息，如果没有新消息，就一直等待。当有新消息的时候，才会返回给客户端。在某种程度上减小了网络带宽和 CPU 利用率等问题。但是长轮询也是存在问题的，由于 http 数据包的头部数据量往往很大（通常有 400 多个字节），但是真正被服务器需要的数据却很少（有时只有 10 个字节左右），这样的数据包在网络上周期性的传输，难免对网络带宽是一种浪费。

## webScoket

### 简介

webScoket是一种网络传输协议，他是基于TCP的全双工通信，同样位于应用层。webScoket是的服务端和客户端长期建立连接并且相互发送消息变成可能。WebSocket 协议是 HTML5 中的一部分，它复用了 HTTP 协议上握手协议来实现双向通信，同时他的端口设计和http一样，都是默认的80和443端口，这样可以使它和http实现兼容。WebSocket 协议可以在现代浏览器和服务器上使用，它使得开发实时 Web 应用程序变得更加容易和高效。

下面这种图就是webScoket和http轮询的区别

![preview](http://gicgo-images.oss-cn-shanghai.aliyuncs.com/img/view.png)

## 建立连接

webScoket的主要有`close`,`error`,`message`,`open`四个事件，具体的使用方法是

```js
//client 
// 创建 WebSocket 连接
const socket = new WebSocket('ws://localhost:8080');

// 连接成功时触发的回调函数
socket.onopen = () => {
  console.log('Connected to server!');
  
  // 向服务端发送消息
  socket.send('Hello, server!');
};

// 接收到服务端消息时触发的回调函数
socket.onmessage = event => {
  console.log(`Received message: ${event.data}`);
  
  // 关闭连接
  socket.close();
};

// 连接关闭时触发的回调函数
socket.onclose = event => {
  console.log(`Connection closed with code ${event.code}`);
};

```

服务端

```js
const WebSocket = require('ws');

// 创建 WebSocket 服务器
const server = new WebSocket.Server({ port: 8080 });

// 连接成功时触发的回调函数
server.on('connection', socket => {
  console.log('Client connected!');
  
  // 接收到客户端消息时触发的回调函数
  socket.on('message', message => {
    console.log(`Received message: ${message}`);
    
    // 向客户端发送消息
    socket.send('Hello, client!');
  });
  
  // 连接关闭时触发的回调函数
  socket.on('close', () => {
    console.log('Client disconnected!');
  });
});

```

## 在浏览器发送请求



类似于 HTTP 和 HTTPS，ws 相对应的也有 wss 用以建立安全连接，本地已 ws 为例。这时的请求头如下：

```yaml
Accept-Encoding: gzip, deflate, br
Accept-Language: zh-CN,zh;q=0.9
Cache-Control: no-cache
Connection: Upgrade	// 表示该连接要升级协议
Cookie: _hjMinimizedPolls=358479; ts_uid=7852621249; CNZZDATA1259303436=1218855313-1548914234-%7C1564625892; csrfToken=DPb4RhmGQfPCZnYzUCCOOade; JSESSIONID=67376239124B4355F75F1FC87C059F8D; _hjid=3f7157b6-1aa0-4d5c-ab9a-45eab1e6941e; acw_tc=76b20ff415689655672128006e178b964c640d5a7952f7cb3c18ddf0064264
Host: localhost:9000
Origin: http://localhost:9000
Pragma: no-cache
Sec-WebSocket-Extensions: permessage-deflate; client_max_window_bits
Sec-WebSocket-Key: 5fTJ1LTuh3RKjSJxydyifQ==		// 与响应头 Sec-WebSocket-Accept 相对应
Sec-WebSocket-Version: 13	// 表示 websocket 协议的版本
Upgrade: websocket	// 表示要升级到 websocket 协议
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_14_0) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/76.0.3809.132 Safari/537.36
```



响应头如下：

```makefile
Connection: Upgrade
Sec-WebSocket-Accept: ZUip34t+bCjhkvxxwhmdEOyx9hE=
Upgrade: websocket
```



![image-20230508173137611](http://gicgo-images.oss-cn-shanghai.aliyuncs.com/img/image-20230508173137611.png)

此时响应行（General）中可以看到状态码 status code 是 101 Switching Protocols ， 表示该连接已经从 HTTP 协议转换为 WebSocket 通信协议。 转换成功之后，该连接并没有中断，而是建立了一个全双工通信，后续发送和接收消息都会走这个连接通道。

注意，请求头中有个 Sec-WebSocket-Key 字段，和相应头中的 Sec-WebSocket-Accept 是配套对应的，它的作用是提供了基本的防护，比如恶意的连接或者无效的连接。Sec-WebSocket-Key 是客户端随机生成的一个 base64 编码，服务器会使用这个编码，并根据一个固定的算法：

```ini
GUID = "258EAFA5-E914-47DA-95CA-C5AB0DC85B11";    //  一个固定的字符串
accept = base64(sha1(key + GUID));	// key 就是 Sec-WebSocket-Key 值，accept 就是 Sec-WebSocket-Accept 值
```



其中 GUID 字符串是 [RFC6455](https://link.juejin.cn?target=https%3A%2F%2Ftools.ietf.org%2Fhtml%2Frfc6455%23section-5.5.2) 官方定义的一个固定字符串，不得修改。

客户端拿到服务端响应的 Sec-WebSocket-Accept 后，会拿自己之前生成的 Sec-WebSocket-Key 用相同算法算一次，如果匹配，则握手成功。然后判断 HTTP Response 状态码是否为 101（切换协议），如果是，则建立连接，大功告成。



## 保持心跳

在实际使用 WebSocket 中，长时间不通消息可能会出现一些连接不稳定的情况，这些未知情况导致的连接中断会影响客户端与服务端之前的通信，

为了防止这种的情况的出现，有一种心跳保活的方法：客户端就像心跳一样每隔固定的时间发送一次 ping ，来告诉服务器，我还活着，而服务器也会返回 pong ，来告诉客户端，服务器还活着。ping/pong 其实是一条与业务无关的假消息，也称为心跳包。

心跳包可能就是一个定时器，发送一个消息，让双方都知道对方服务还活着
