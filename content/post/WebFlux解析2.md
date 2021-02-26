---
author: "bufferflies"
date: 2020-11-04
title: "webflux源码设计2"
tags: [
    "webflux",
]
categories: [
    "spring"
]
---

## 核心类图
```plantuml
class RouterFunction {
  RouteFunction routeFunction; 路由执行器
  List<HttpMessageReader<?>> messageReaders; request阅读器
  getHandlerInternal(ServerWebExchange ex): HandlerFunction 获取对应的执行器
}

class HandlerFunctionAdapter {
  supports(Object handler): boolean 只支持HandlerFunction 的函数
  handler(ServerWebExchange ex,Object handler): HandlerResult 执行方法并进行的转换
}

class ServerResponseResultHandler {
    List<HttpMessageWriter<?>> messageWriters: response 编写器
    List<ViewResolver> viewResolvers： 视图映射
    supports(HandlerResult result)：boolean 只处理返回类型为ServerResponse类型
    handleResult(ServerWebExchange exchange,HandlerResult result): void 将结果写入到response中
    
}

interface HandlerFunction{
   handle(ServerRequest request): T 最直接的操作 业户逻辑入口 一般lambda实现
}

interface RequestPredicate{
    test(ServerRequest request): boolean
}

interface RouteFunction {
  route(ServerRequest request): HandlerFunction 该路由是否能够满足该请求 核心
  and(RouteFunction other):  SameComposedRouterFunction 合并
  andOther(RouteFunction other): DifferentComposedRouterFunction 合并
  andRoute(RequestPredicate predicate, HandlerFunction<T> handlerFunction): RouterFunction 比较底层
  andNest(RequestPredicate predicate, RouterFunction<T>  routerFunction): RouterFunction 合并路由
  filter(HandlerFilterFunction<T, S> filterFunction): 带有过滤的路由
}
```

解释：
1. RouteFunction 会将所有的RouteFunction进行聚合 
2. 过滤其实是在mapping的时候执行的

初始化过滤：

## 时序图

```plantuml
actor Processor as U
control RouteFunction as R
control Predicate as P 
control HandlerFunctionAdapter as A
control ServerResponseResultHandler as R

U -> R : getHandler
R -> P : test the request
P -->R : OK
R -->U : return HandlerFunction
U -> A : execute the handler
A -->U : return and wrap to HandlerResult
U -> R: test
R ->U : write value to response  
```
 





