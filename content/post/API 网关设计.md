---
author: "bufferflies"
date: 2021-02-26 
title: "API 网关设计"
tags: [
    "go"
]
categories: [
    "gateway"
]
---

# 网关评审
## 需求
需要的能力：
- 路由
- 协议转换
- 服务发现
- mertices监控
- 路由动态更新
- 支持OAuth2，LDAP身份认证
- 熔断
- 重试
- 流量控制
- 负载均衡

### 场景一
请求直接到后端服务
```plantuml
node request

node services1
node services2

request-->services1
request-->services2
```

### 场景二

```plantuml
node request

node gateway
node consul

node services1
node services2

request-->gateway: send request
gateway->consul: send server name 
consul--> gateway: return ip table

gateway-->services1
gateway-->services2
```

网关主要的功能：
- 对请求进行鉴权 这一块业务性质比较重
- 统计请求耗时，加监控，业务性质比较重
- 匹配请求，选择最合适的路由进行转发

因此主要的功能：
- 能够自定义全局过滤器，适用到所有请求 ，如监控
- 能够自定义断言器，根据请求来决定其访问的服务
- 能够自定义过滤器，配合断言器，适用于指定的请求


## Janus设计

```plantuml
storage predicate {
    storage PredicateFactory
    storage PathPredicateFactory
    PredicateFactory<--PathPredicateFactory
}
storage route {
     storage id
     storage uri 
     storage Predicate
     storage FilterChain
     storage Filter
     id-->uri
     id-->Predicate
     id-->FilterChain
     FilterChain-->Filter: filter
}
storage WebHandler{
    card Routes
}
storage filter {
    storage FilterFactory
    storage RemoveFilterFactory 
    FilterFactory<--RemoveFilterFactory
}
control request
Filter<--FilterFactory:factory
Predicate<--PredicateFactory: factory
request ->WebHandler
WebHandler->route
```


## 后期的想法
### 依靠trpc，所有服务暴露多种协议
```plantuml
node request

node gateway
node consul

storage {
   card container1
   node services1
   container1->services1
}
storage {
   card container2
   node services2
   container2->services2
}
node services2

request-->gateway: send request
gateway->consul: send server name 
consul-> gateway: return ip table

gateway-->container1
gateway-->container2
```
container 作为容器 提供各种协议
service 专注于业务 容器会将socket转换到相应的入参


### 在网关上进行配置 指定协议
```plantuml
node http_request

node gateway
node consul

node http_proxy
node grpc_proxy

node services1
node services2

http_request-->gateway: send request
gateway->consul: send server name 
consul-> gateway: return ip table

gateway-->http_proxy: proxy
gateway-->grpc_proxy: proxy

http_proxy-->services1: handle
grpc_proxy-->services2: handle

```

grpc_proxy 提供Http转rpc功能
http——proxy 提供http代理 
