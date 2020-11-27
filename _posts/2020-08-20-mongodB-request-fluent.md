---
layout: post
title: "MongoDB-University-Elvis-请求处理流程"
subtitle: 'mongo'
author: "Elvis"
header-style: text
tags:
  - mongo
  - 技术分享
---



mongodb处理客户端请求的模型



Mongod在启动时会调用createServer创建一个PortMessageServer对象，其继承MessageServer和Listener两个类，并依赖MyMessageHandler来处理请求。

### PortMessageServer

那PortMessageServer主要完成了什么功能？

![img](/img/in-post/post-mongo/port_message_server.png)

主要为以下三点：

1. 为mongod配置的每个地址创建一个socket，并调用bind绑定地址。
2. 调用initAndListen监听所有的地址，调用select等待监听fd上发生连接事件
3. 调用accept系统调用接受新的连接请求，并为每个新连接创建一个线程，该线程执行handleIncomingMsg方法，不断处理该连接上的客户端请求。



### handleIncomingMsg

那handleIncomingMsg怎么调用Mongo方法处理请求的呢？

![img](/img/in-post/post-mongo/handle_incoming_msg.png)

1. 连接建立时，调用MyMessageHander::connected方法，初始化一个新的Client对象，Client对象包含DB操作的上下文。
2. 不断调用recv从连接上读取请求，当读取到一个完整请求时，其将请求反序列化为一个Message对象，并调用MyMessageHandler::process方法处理请求，处理完后给客户端发送应答。
3. 连接断开时，调用MyMessageHander::disconnected方法停止该连接对应的线程，释放Client对象。

