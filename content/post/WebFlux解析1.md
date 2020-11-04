---
author: "bufferflies"
date: 2020-11-04
title: "webflux源码设计1"
tags: [
    "java",
    "spring"
]
categories: [
    "webflux"
]
---

# Spring5 WebFlux 源码解析（一）

## 场景：
 能够将**请求**适配到**指定方法**，**调用**方法并**返回**

## 领域图

```plantuml
storage Request as W
storage HandlerMapping as M
storage HandlerAdapter as A
storage HandlerResultHandler as R
entity Result as E
control Handler as C

W -> M: trigger
M -> C: find 
E -> R: consume 
C -> A: adapt
A -> E: transfer

```

对象解释
 Request: 代表用户的请求
 HandlerMapping： 处理器寻找器 根据请求参数，寻找到处理器
 HandlerAdapter： 处理器适配器（入参和出参适配）
 HandlerResultHandler：结果处理器
 Result： 处理器处理后的结果
 Handler： 处理器（需要用户编写）
 

## 类图

```plantuml
interface HandlerMapping{
    getHandler(ServerWebExchange ex):Object
}
interface HandlerAdapter{
    supports(Object object):boolean
    handler(ServerWebExchange ex, object):HandlerResult
}
interface HandlerResultHandler{
    supports(HandlerResult result): boolean 
    handle(ServerWebExchange ex, HandlerResult result): void
}
class HandlerResult{
    Object handler
    Object returnType
    Object returnValue
    Object returnBingContext
    Function<Throwable, HandleResult> exceptionException
}
```

## 时序图
```plantuml
actor User as U
boundary DispatcherHandler as D
control HandlerMapping as M
control HandlerAdapter as A
control HandlerResultHandler as R

U -> D: send Request
D -> M: find a handler that can resolve the request
M --> D: return Handler
D -> A: process the handler
A --> D: return HandlerResult
D -> R: process result 
R --> D: return response 
D --> U: render  
```


## 核心链式调用：
```plantuml
(*) --> [request] "HandlerMapping"
-->[Handler] "HandlerAdapter"
-->[Adapt] "HandlerResult"
-->[Response] "Socket"
-->(*)
```

### 附录
#### 附录一
spring 最核心代码
```c 
	public Mono<Void> handle(ServerWebExchange exchange) {
		if (this.handlerMappings == null) {
			return createNotFoundError();
		}
		// 找到所有注册的Mappings
		return Flux.fromIterable(this.handlerMappings)
				// 按序寻找可以处理该请求的Mapping
				.concatMap(mapping -> mapping.getHandler(exchange))
				// 只返回一个
				.next()
				//无法进行处理 抛出404异常
				.switchIfEmpty(createNotFoundError())
				// 寻找最适合的适配器并执行Handler 返回结果
				.flatMap(handler -> invokeHandler(exchange, handler))
				// 寻找结果处理器， 写回到Response中
				.flatMap(result -> handleResult(exchange, result));
	}
```
 核心类单元测试构造
 1. 映射处理器

 ```java
 // mock 一个HandlerMapping 接口 并使其继承Ordered 接口
 HandlerMapping hm1 = mock(HandlerMapping.class, withSettings().extraInterfaces(Ordered.class));
 // 实现Ordered接口
given(((Ordered) hm1).getOrder()).willReturn(1);
// 给定任何参数返回“1”
given((hm1).getHandler(any())).willReturn(Mono.just((Supplier<String>) () -> "1"));
 ```
 1. 适配器

 ```java
 
 private static class SupplierHandlerAdapter implements HandlerAdapter {

    // 支持所有的Supplier类型的处理器
		@Override
		public boolean supports(Object handler) {
			return handler instanceof Supplier;
		}
   // 执行处理器 将其结果转换为HandlerResult
		@Override
		public Mono<HandlerResult> handle(ServerWebExchange exchange, Object handler) {
			return Mono.just(new HandlerResult(handler, ((Supplier<?>) handler).get(), RETURN_TYPE));
		}
	}
 ```
 2. 结果处理器

 ```java
 private static class StringHandlerResultHandler implements HandlerResultHandler {

		@Override
		public boolean supports(HandlerResult result) {
			Object value = result.getReturnValue();
			return value != null && String.class.equals(value.getClass());
		}

		@Override
		public Mono<Void> handleResult(ServerWebExchange exchange, HandlerResult result) {
		// 获取response中的值
			byte[] bytes = ((String) result.getReturnValue()).getBytes(StandardCharsets.UTF_8);
			// 将其转换为DataBuffer类型
			DataBuffer dataBuffer = new DefaultDataBufferFactory().wrap(bytes);
			// 填充到response中
			return exchange.getResponse().writeWith(Mono.just(dataBuffer));
		}
	}
 ```