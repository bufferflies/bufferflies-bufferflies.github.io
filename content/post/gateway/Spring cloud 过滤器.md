---
author: "bufferflies"
date: 2021-02-26 
title: "Spirng Cloud 过滤器"
tags: [
    "spring",
]
categories: [
    "gateway"
]
---

# 网关
## 通用过滤器
 
| 过滤器 | 作用 | 描述 |
| --- | --- | --- |
| RemoveCachedBodyFilter | 处理缓存 | MIN_LEVEL |
| AdaptCachedBodyGlobalFilter |  | MIN_LEVEL+1000 |
| NettyWriteResponseFilter | 返回response body | -1 |
| ForwardPathFilter | 处理uri为"forward"请求 | 0 |
| RouteToRequesrUrlFilter | 处理URI 替换原有的Request的URI| 10000 |
| NoLoadBalancerClientFilter | 判断URL中不含有lb前缀 | 10100 |
| WebsocketRoutingFilter |  | MAX_LEVEL-1 |
| NettyRoutingFilter | 开始请求 | MAX_LEVEL |
| ForwarRoutingFIlter |  | MAX_LEVEL |
