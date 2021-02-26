---
author: "bufferflies"
date: 2021-02-26 
title: "spring 源码解析--mvc 处理流程"
tags: [
    "spring mvc",
]
categories: [
    "spring"
]
---


# spring 源码解析--mvc 处理流程
客户端发出HTTP请求，框架找到并执行对应的方法，最后将执行的结果进行返回

简单的版本预设条件：
    1.不用实现过滤器，
    2.只允许Ajax请求
## 架构
关系

``` plantuml
component request
frame UrlMapping{
   storage SimpleUrlHandlerMapping  
} 

frame ControllerAdapter{
    storage HandleAdapter
}
frame ResultHandler{
  storage ModelView
}
request ->UrlMapping
```
UrlMapping：根据request的请求找到执行器chain
ControllerAdapter: 执行器的适配器
ResultHandler：处理执行器的结果
类图

``` plantuml
 class HandlerExecutionChain{
   Object handler: Controller 也就是我们自己的写的controller
   List<HandlerInterceptor> interceptorList: 拦截器列表
   int interceptorIndex： 拦截器执行的索引
   applyPreHandle(request，response):boolean 执行前置拦截器
   applyPostHandler(request, response): boolean 执行后置拦截器 
 }
 class HandlerAdapter{
    supports():boolean 给定一个handler 判断其是否可以处理
    handler(request,response，handler):ModelView 执行方法
    getLastModified(request, handler): long
 }
 
 class HandlerInterceptor{
   preHandler(request,response，handler):boolean
   postHandler(request,response，handler):boolean
   afterCompletion(request,response，handler):boolean after rendering the view
 }
```
HandlerExecutionChain 执行器
HandlerAdapte  handler的适配器 对controller进行管控
HandlerInterceptor 拦截器父类


时序图

``` plantuml
    actor User as U
    boundary WebContainer as C
    control DispatcherServlet as D
    control HandlerExecutionChain as E
    control HandlerAdapter as A
    control HandlerIntercepter as I
    
    U->C: send request
    C->D: doService
    D-> E: getHandler(request)
    E-->D: return chain
    D->A: getHandlerAdapter(chain)
    A-->D: return HandlerAdapter
    D-> D: HandlerExecutionChain.applyPreHandler \n 开始执行前置拦截器
    D-> D: HandlerAdapter.handler mv \n 开始执行Controller中的方法
    D-> D: applyDefaultViewName(request, mv) \n 返回默认的视图
    D-> D: applyPostHandler(result, response, mv) \n 执行后置拦截器 
    D->D: processDispatchResult \n (request, response, mappedHandler, mv, dispatchException)\n 处理返回的结果
    D->D: applyAfterConcurrentHandlingStarted(request, response) \n 执行拦截器的completion方法
    
``` 