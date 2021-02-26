---
author: "bufferflies"
date: 2021-02-26 
title: "一次OkHTTP3 网络排查"
tags: [
    "java",
    "OkHttp"
]
categories: [
    "net"
]
---

# 记一次http pipeline异常
## 背景
### 服务依赖关系
```plantuml
storage Client{
    storage OkHttp
    storage Netty
}
storage Server{
    storage Control 
    storage Tomcat
}

OkHttp->Netty
Netty.>Tomcat
Tomcat -> Control
```

### 现象
每天偶现5次左右API调用返回400现象, 查询Server端日志：

```shell
[DEBUG] [io.undertow.server.protocol.http.HttpReadListener sendBadRequestAndClose 284] - UT005014: Failed to parse request i in request-targetdRequestException: UT000165: Invalid character 
        at io.undertow.server.protocol.http.HttpRequestParser.handlePath(HttpRequestParser.java:382) ~[undertow-core-1.4.26.Final.jar!/:1.4.26.Final]
        at io.undertow.server.protocol.http.HttpRequestParser.handle(HttpRequestParser.java:248) ~[undertow-core-1.4.26.Final.jar!/:1.4.26.Final]
        at io.undertow.server.protocol.http.HttpReadListener.handleEventWithNoRunningRequest(HttpReadListener.java:187) [undertow-core-1.4.26.Final.jar!/:1.4.26.Final]
        at io.undertow.server.protocol.http.HttpReadListener.handleEvent(HttpReadListener.java:136) [undertow-core-1.4.26.Final.jar!/:1.4.26.Final]
        at io.undertow.server.protocol.http.HttpReadListener.handleEvent(HttpReadListener.java:59) [undertow-core-1.4.26.Final.jar!/:1.4.26.Final]
```

## 排查流程
根据日志异常提示，报文中存在非法字符，两方面排查:
1. 服务端容器对HTTP协议解析有问题
2. 客户端发过来的请求存在非法字符 

首先尝试换容器，使用tomcat替换undertow， 线上运行测试，发现问题依然存在，开始抓包

### 抓包
抓包结果如下图，出现6次400请求：
![-w1040](/网络排查1.jpg)

使用tcp follow 查看一下报文，并没有发现异常情况：
![-w981](/网络排查2.jpg)

具体查看一下报文：
![-w1513](/网络排查3.jpg)

**查看上面报文发现请求中携带content-length**

继续看看body中的内容：

![-w803](/网络排查5.jpg)

发现content中的是下一个http 请求（请求B）， 请求B的一部分被放到上一个请求中，导致Server端无法解析请求B，报出非法字符，返回400 请求

继续查看一下http follow情况，如下图， 请求B的一部分：
![-w996](/网络排查6.jpg)

验证之前的分析，这个请求不是Pipeline，正常的pipeline格式如下：

```shell
echo -en "GET /index.html HTTP/1.1\r\nHost: example.com\r\n\r\nGET /other.html HTTP/1.1\r\nHost: example.com\r\n\r\n" | nc example.com 80
```

## 处理方法
1. 升级OKHttp到4.0 
2. 不使用OKHttp ，使用HttpComponent
