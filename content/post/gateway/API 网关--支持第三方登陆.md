---
author: "bufferflies"
date: 2021-02-26 
title: "API 网关设计 OAuth2"
tags: [
    "go",
    "OAuth2"
]
categories: [
    "gateway"
]
---


# 领域模型
核心功能：为网关提供用户认证服务
提供的功能：
- 能够接入支持OAuth2的认证服务
- 用户登陆成功后可以跳转到之前访问的页面

## 领域图

```plantuml

storage request
storage proxy 
storage token
storage identification

rectangle verify{
    storage Verify 
}
rectangle signature{
    storage Signature 
}
rectangle Jwt{ 
}
rectangle auth{
    storage Auth 
    storage OAuth2
    OAuth2->Auth 
}

request <->proxy: proxy
request ..> token:carry
token --> Verify: use
identification <-- Verify: verify

identification <--Auth : consult 


identification -->Signature: use
token <--Signature: sign
Verify<--Jwt
Signature<--Jwt
```


Signature: 负责加签，
Verify: 负责解签，可以获取加签之前的内容
Auth：获取用户信息

identify： 用户信息
token： 加签过后的凭证


## 类图

```plantuml

interface Signature{
    sign(Object signature): String
}
interface Verify{
    verify(String token): Object
}

interface Auth{
    consult(String code):Object
}

interface State{
    encode(URL url): String code
    decode(String code): URL url 
}

class SimpleState{
    - Duration expired
    - Integer size 
    encode(URL url): String code
    decode(String code): URL url
}

class JwtSignature {
     - SignatureAlgorithm alg
     - Duration expired
     - String issue
     - String subject
     sign(Object signature): String
     verify(String token): Object
}

class OAuth2{
    - String urlTemplate
    - RestTemplate client
    consult(String code):Object
}

Signature <-- JwtSignature
Verify <-- JwtSignature
State <-- SimpleState
Auth<--OAuth2
```

## 时序图

```plantuml

actor User as U
control Gateway as G
control AuthFilter as A
control UserAction as AC
control Proxy as P
control service as S 
control OAuth2 as O

== request with legal token ==
U -> G: send xxx.weling.com/auth/xxx
G -> A: validate token
A --> G: token is legal
G -> P: send auth.weling.com/xxx 
P -> S: handler
S --> P: return result
P --> G: post handler
G --> U: render 
== request with nothing ==

U -> G : request xxx.weling.com/auth/xxx
G -> A : verify token
A -> G : token is illegal
G -> O : 302 open.weixin.qq.com? \n appid=xxx\n&redirect_url=xxx.weling.com/consult/ xxx \n&scope=xxx\n&state=xxx.weling.com/auth/xxx
O ->O : wait user login 
O -->G : 302  xxx.weling.com/consult&code=xxx&\nstate=xxx.weling.com/auth/xxx
G -> O : consult identification \nopen.weixin.qq.com\n&appid=xxx\n&code=xxx\n&secret=xxx
O -->G : return identify 
G -->U : return redirect html \n and set cookie
U ->U : set cookie and redirect
```