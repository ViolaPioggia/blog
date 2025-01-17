---
title: "简单探索 SSE 网络协议"
date: "2023-12-10"
---

## 是什么

这种通信方式并非传统的 http 接口或者 WebSockets，而是基于 EventStream 的事件流，像打字机一样，一段一段的返回答案。GPT就是采取的这种网络协议

![](images/image.png)

Server-Sent Events 服务器推送事件，简称 SSE，是一种服务端实时**主动**向浏览器推送消息的技术。

​ SSE 是 HTML5 中一个与通信相关的 API，主要由两部分组成：服务端与浏览器端的通信协议（`HTTP` 协议）及浏览器端可供 JavaScript 使用的 `EventSource` 对象。

​ 从“服务端主动向浏览器实时推送消息”这一点来看，该 API 与 WebSockets API 有一些相似之处。但是，该 API 与 WebSockers API 的不同之处在于：

![](images/image-1.png)

## 实现

SSE 的实现相对比较简单（指和其他网络协议相比），因为它是基于 HTTP 的在应用层上的协议

本质是浏览器发起 http 请求，服务器在收到请求后，返回状态与数据，并附带以下 headers：

`Content-Type: text/event-stream`

`Cache-Control: no-cache`

`Connection: keep-alive`

SSE API规定推送事件流的 MIME 类型为 `text/event-stream`。必须指定浏览器不缓存服务端发送的数据，以确保浏览器可以实时显示服务端发送的数据。SSE 是一个一直保持开启的 TCP 连接，所以 Connection 为 keep-alive。

### 消息格式

EventStream（事件流）为 `UTF-8` 格式编码的`文本`或使用 Base64 编码和 gzip 压缩的二进制消息。

​ 每条消息由一行或多行字段（`event`、`id`、`retry`、`data`）组成，每个字段组成形式为：`字段名:字段值`。字段以行为单位，每行一个（即以 `\n` 结尾）。以`冒号`开头的行为注释行，会被浏览器忽略。

​ 每次推送，可由多个消息组成，每个消息之间以空行分隔（即最后一个字段以`\n\n`结尾）。

### 客户端实现

客户端实现其实也比较简单，只要设置好 URL ，相应的一些 Header，然后自定义一些方法就能实现 SSE 客户端了。要实现一些比较强大的功能比如说断线重连、传输大文件、通道传输等等就需要在服务端多思考

## 总结

​ SSE 可以在 Web 应用程序中实现诸如股票在线数据、日志推送、聊天室实时人数等即时数据推送功能。需要注意的是，SSE 并不是适用于所有的实时推送场景。在需要高并发、高吞吐量和低延迟的场景下，WebSockets 可能更加适合。而在需要更轻量级的推送场景下，SSE 可能更加适合。因此，在选择即时更新方案时，需要根据具体的需求和场景进行选择。
