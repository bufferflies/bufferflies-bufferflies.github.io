---
author: "bufferflies"
date: 2021-02-26 
title: "API 网关设计 插件编写"
tags: [
    "go"
]
categories: [
    "gateway"
]
---

# trpc/http-gateway
需求：对外提供HTTP的转发功能
特性：
 - 支持自定义的Filter和Predicate
 - 兼容spring cloud gateway的配置

 
## 网关设计
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
执行：
1. 请求送达到WebHandler中，WebHandler寻找能够处理该请求的Route
2. WebHandler找到这个Route，执行过滤器链


名词解释：
- Route： 一个代理的抽象 包含断言和过滤器
- FilterChain： 过滤器执行器
- Filter： 过滤器，处理http请求
- Predicate： 断言器，给定一个请求，判断其是否可以处理





说明：
 1. Webhandler : 请求处理器，负责这个请求的执行
 2. FilterFactory： 为过滤器的工厂
 3. PredicateFactory：为断言器的工程
 4. route：一个代理的设置
     - id： 一个代理的唯一ID
     - uri： 最终的uri
     - predicate：使用的Http断言器
     - filters：设置的过滤器
     
## 类图
```plantuml
interface Filter{
    PreHandler(request, response)bool
    PostHandler(request, response)
}

class RequestFilter{
    PreHandler(request, response)bool
    PostHandler(request, response)
    int order
}

class GlobalFilter{
    PreHandler(request, response)bool
    PostHandler(request, response)
    int order
}

class FilterChain{
    FilterChain(Filter filters[])
    Filter(request, response)
}

class RequestFilterFactory<T>{
    New(string config)RequestFilter
}
interface RequestPredicate{
    Test(request)bool
}
class Route{
    string id
    string url
    RequestPredicate predicate
    RequestFilter []filters
    
}
class WebHandler{
    Route route[]
    WebHandler(string path)
    ServerHttp(request, response)
    Lookup(request)Route   
}


class RequestPredicateFactory<T>{
    New(T t)RequestPredicate
}
Filter <.. RequestFilter 
Filter<..GlobalFilter
RequestPredicate <-- RequestPredicateFactory
WebHandler<--*Route
Filter<-- RequestFilterFactory
```
约定：
    方法大写为public方法 
    属性名大写为Public变量

## 时序图
```plantuml
actor Request as R
control WebHandler as W
control Route as T
control Predicate as P
control FilterChain as C
control Filter as F

R -> W: request xxx.weling.com/test
W -> T: consult
T -> P: do test method 
P -->T: return boolean 
T -> C: do filter 
C -> F: do PreHandler and PostHandler
F -->C: return response
C -->T: complete
T -->W: write response to socket
W --> R: render
```

## 使用说明
##程序启动
使用步骤：
1. 配置trpc——go.yml 文件
2. 配置路由配置
3. 启动

启动类
```golang
package main

import (
	"fmt"
	"gateway/handler"
	web "git.code.oa.com/mossli/trpcweb"
	"git.code.oa.com/trpc-go/trpc-go"
	"git.code.oa.com/trpc-go/trpc-go/filter"
	"git.code.oa.com/trpc-go/trpc-go/log"
	"git.code.oa.com/trpc-go/trpc-go/plugin"
	"git.code.oa.com/trpc-go/trpc-go/server"
	"net/http"
)
func main(){

	// load log config
	plugin.Register("custom", log.DefaultLogFactory)
	// load trpc config 
	opts := []server.Option{
		server.WithFilter(filter.NoopFilter),
	}
	server := trpc.NewServer(opts...)
	s := web.Server(server)
	// for gateway
	svi := web.Service(s.Server, "web.mossli.gateway.TestSrv2")
	h:=handler.New()
	svi.Handle("/",h)
	// start stub for test
	svi2 := web.Service(s.Server, "web.mossli.gateway.TestSrv3")
	svi2.HandleFunc("/",func(w http.ResponseWriter, r *http.Request){
		w.WriteHeader(200)
		fmt.Fprintf(w, "echo")
	})
	s.Serve()
}
```

路由配置
```json
[{
  "id": "auth",
  "uri": "lb://localhost:8021",
  "predicates": [
    {
    //为自定义断言器工厂的name
      "name": "Path",
      "args": "/test/*"
    }
  ],
  "filters": [
    {
    //为自定义过滤器工厂的name
      "name": "RemoveHeader",
      "args": "Auth"
    }
  ]
}]
```


### 自定义断言器（Predicate）
断言器是**多例的**，但是其工厂是**单例的**
约定：
- 断言器命名以Predicate结尾
- 断言器工厂命名以PredicateFactory结尾

步骤：
1. 自定义断言器 实现RequestPredicate接口
2. 自定义断言器工厂 实现RequestPredicateFactory
3. 推荐自定义一个Config结构体
4. 注册断言器工厂

代码如下：
```golang
package predicate

import (
	"net/http"
	"strings"
)

func init(){
	RegisterFactory("method", &MethodPredicateFactory{})
}
/**
请求方法Method断言器
 */
type MethodPredicateFactory struct {
}
type MethodRequestPredicate struct{
	config MethodConfig
}

type MethodConfig struct {
	methods []string
}
func (p *MethodRequestPredicate)Test(r *http.Request)bool{
	for _,method:=range p.config.methods{
		if method==r.Method{
			return true
		}
	}
	return false
}
// methods 以逗号进行分割
func (p *MethodPredicateFactory)New(methods string)RequestPredicate{
	m:=strings.Split(methods,",")
	// http 中method均是大写
	for i,method:=range m{
		m[i]=strings.ToUpper(method)
	}
	return &MethodRequestPredicate{MethodConfig{m}}
}
```

### 自定义过滤器工厂
过滤器是**多例的**，但是其工厂是**单例**

约定
- 过滤器命名以Filter结尾
- 过滤器工厂命名以FilterFactory结尾

使用步骤：
1. 自定义过滤器 实现Filter接口
2. 自定义过滤器工厂 实现FilterFactory
3. 推荐自定义一个Config结构体
4. 注册过滤器工厂


代码如下
```golang
package filter

import (
	"gateway/utils"
	"net/http"
	"strings"
)

func init(){
	Register("RemoveHeader",&RemoveHeaderFilterFactory{})
}
type RemoveHeaderFilterFactory struct{
}

type RemoveHeaderFilter struct{
	config RemoveConfig
	order int
}
type RemoveConfig struct {
	headers []string
}

func (f *RemoveHeaderFilter) PreHandler(response http.ResponseWriter,request *http.Request)bool{
	headers:=request.Header
	for name,_:=range headers{
		if utils.ContainerString(f.config.headers,name){
			headers.Del(name)
		}
	}
	return true
}
func (f *RemoveHeaderFilter) PostHandler(response http.ResponseWriter,request *http.Request){

}
func (f *RemoveHeaderFilter)getOrder()int{
	return f.order
}
func (f *RemoveHeaderFilter)SetOrder(order int)Filter{
	f.order=order
	return f
}
func (f *RemoveHeaderFilterFactory)New(config string)Filter{
	return &RemoveHeaderFilter{RemoveConfig{strings.Split(config,",")},0}
}
```

### 自定义全局过滤器
有时候需要使用全局过滤器，针对的是所有的代理，全局代理器是**单例**

约定：
- 全局过滤器以GlobalFilter结尾

使用步骤：
1. 自定义过滤器 实现Filter接口
2. 注册全局过滤器

```golang
package filter

import (
	"fmt"
	"net/http"
	"net/http/httputil"
	"net/url"
	"trpc-go/log"
)

/**
进行注册
 */
func init(){
	RegisterGlobalFilter("proxy",&ProxyFilter{order:1024})
}
type ProxyFilter struct{
	order int
}
func (p *ProxyFilter) PreHandler(response http.ResponseWriter,request *http.Request)bool{
	defer func(){
		if err:=recover();err!=nil{
			log.Error("proxy filter has error",err)
		}
	}()

	route:=request.Context().Value("route")
	if route!=nil{
		fmt.Printf("uri is %s /n",route)
	}else {
		return false
	}
	r := route.(string)
	uri,err:=url.Parse(r)
	if err!=nil{
		response.WriteHeader(http.StatusInternalServerError)
		return false
	}
	// not support http and https
	if uri.Scheme!="http"||uri.Scheme!="https"{
		response.WriteHeader(http.StatusInternalServerError)
		return false
	}
	proxy := httputil.NewSingleHostReverseProxy(uri)
	proxy.ServeHTTP(response, request)
	return true
}
func (p *ProxyFilter) PostHandler(response http.ResponseWriter,request *http.Request){
	fmt.Println("proxy complete")
}
func (p *ProxyFilter) getOrder()int{
	return p.order
}
func (p *ProxyFilter) SetOrder(order int)Filter{
	p.order=order
	return p
}
```
 
 