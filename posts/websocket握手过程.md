---
title: websocket握手过程
date: 2018-08-11 14:40:56
tags: [http]
---

websocket是是一种客户端与服务端全双工通信的一种方式，它基于http，借用http中的某些字段来进行握手的，不同的浏览器实现不同，但是大体有一些关键的字段，讲一讲他们的握手过程。
<!--more-->
# client端
首先client端会发送以下信息
```
Connection:Upgrade #通知服务器协议升级
Upgrade:websocket  #协议升级为websocket协议
Host:0.0.0.0:9501  #升级协议的服务主机:端口地址
origin:xxxx #用于给服务器判断是否接受握手
Sec-WebSocket-Key:K8o1cNIxO2pR6inTIDBSgg== #传输给服务器的key
Sec-WebSocket-Version:13 #websocket协议版本13
```
# 服务端：
然后是服务端的回复
```
Connection:Upgrade #协议升级成功
Sec-WebSocket-Accept:GnoYH/ip/ZMh+a5rX5P/YR6e68g= #服务端加密后的key，客户端解密后如果与自己发送的key相同，握手才会成功
Sec-WebSocket-Version:13#websocket 协议版本号
Upgrade:websocket#协议升级为websocket
```
之后就是正式的发送数据流程了。