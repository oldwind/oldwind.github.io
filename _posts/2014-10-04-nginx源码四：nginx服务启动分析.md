---
layout: content
title: nginx源码四：nginx服务启动分析
status:  3
complete: 10% 
category: nginx
---

## 一.前言
了解nginx的服务启动，需要了解一个核心问题，就是nginx的架构设计； 要了解架构设计，我想可能需要明白，`为什么要设计nginx？也就是nginx存在的价值`，在我看来，需求定位是架构之母，架构没有最好，只有最合适；实际上任何一门语言都可以很容易的启动一个socket服务，去分析http协议，去处理http请求，例如go语言的gin框架，可以在服务器上简单快速的启动一个server。

我想nginx的出现，或者说是webserver的出现，归根结底从两个方面考虑，一是"共性"， 二是"特性"。

所谓**"共性"**是计算机架构设计的特点，将公共服务独立起来，例如，在大的业务设计中，我们将，"短链服务"，"短信服务"等等独立出来，原因不外乎，一是可以统一管理入口，我们想象如果类似的代码遍布整个系统，那么修改维护的代价就会很高，将nginx这种web服务从业务系统剥离出来，也是抽出共性，统一管理； 原因二，应对外部的变化，http协议从0.9到2.0经历了多次变化，这个变化如果有统一的模块管理，将减少业务系统维护的复杂性

二是**"特性"**；nginx出现后，作为一个"媒介"，对上接收http请求，对下链接各类后端服务, 形成了nginx非常多独有的功能；
- 负载均衡； nginx可以作为一个代理服务器，将上游请求转发到各个下游服务
- 支持https；
- "动静分离"； nginx提供静态资源的直接获取，减少后端服务的开销
- 流量管控； 解决资源访问的安全问题

实际上，nginx作为一个web服务，定位上，是做好一个`中介`的角色，对上解决和上游的对接，处理各种上游交互协议的变化(http,https)； 对下，支持各类协议，将业务交给不同的下游去处理，实际上下游也有多种协议类型；
- http协议
- fastcgi协议
- scgi协议
- cgi协议
- memcache的协议
..

我们可以看出，像nginx这样的web服务器，采用"插件化" 架构是最合适的； 表现在几个方面
- 一是扩展性强，插件化架构，适合编辑各种插件，丰富nginx的生态，例如支持下游协议增加和修改；同时可以编写各类服务插件例如"降级"，"限流"等等
- 二是主流程的数据结构变化比较小

那么下面我们从代码结构开始，看看nginx是怎么实现"插件化"的架构的

## 二. nginx代码结构

nginx的代码结构比较清晰， 核心代码在src文件下

{% highlight bash%} 
├── auto   ## nginx编译工具集
│   ├── cc
│   ├── lib
│   ├── os
│   └── types
├── conf    ## nginx的基础conf文件夹
├── contrib
│   ├── unicode2nginx
│   └── vim
├── html  
├── man    ## nginx的帮助文件
├── objs   ## nginx编译objs
└── src    ## nginx代码库文件夹
    ├── core                ## nginx核心代码，存放nginx的main函数，nginx基础数据结构
    ├── event               ## nginx事件处理模块
    │   └── modules         ## nginx事件处理的"插件集合"，支持 epoll， select/poll， kqueue 等等
    ├── http                ## nginx的http处理模块
    │   └── modules         ## nginx的http"插件"集合
    │       └── perl
    ├── mail    ## nginx邮件处理模块，支持采用nginx做邮件服务器
    ├── misc
    └── os    ## 跨平台封装模块
        └── unix           ## unix的系统API封装      
{% endhighlight %} 


## 三. nginx启动流程
### 3.1 从configure开始

在前面我们说了， nginx采用插件化架构思路，那么系统加载插件的方式一般有两种，
- 一种是动态加载， 例如apache和php的关系，apache把php当成一个插件，php被打成 .so或者.dll 的动态链接库，apache在动态加载，完成调用
- 第二种，像nginx这样，动态生成一个"插件列表"，代码流程中根据"插件列表"选择合适插件

我们看一下插件列表的生成和使用

3.1.1. "插件"开发好后，在configure里面加入

{% highlight bash%} 
./configure --prefix=/Users/baidu/dev/nginx --add-module=/{$USER_PATH}/github/nginx-stability-module/
- --prifix 安转目录
- --add-module 加入第三方模块地址
- --with-http_stub_status_module 加入非默认的第三方库地址
{% endhighlight %} 

configue根据参数，找到"插件"， 在objs目录下会生成一个nginx的"插件"数组 `ngx_module.c`， 这个生成的数组会被





### 3.2 进入main函数


## 四. nginx的"插件化"

### 3.1 从configure开始



