---
layout: content
title: http协议中Transfer-Encoding的chunked
status: 3
complete: 10% 
author:     "yimuren"
tags:
    - http
---

## 一. 前言
厂内ODP框架封装了一个叫RAL的网路通讯工具，解决负载均衡、打包、日志ID透传等一系列功能，RAL是一款PHP的扩展，在和第三方系统交互的时候，我习惯与抓原始包看交互情况，这次遇到一个我没法理解的问题。实际上是针对http协议的。我的理解，http协议的返回应该是这样的

1、协议头
2、两个换行符
3、body体
我在抓包的时候发现，Body体非常的奇怪，示例如下

{% highlight bash %} 
    0x0190:  6163 6563 6f64 653a 2030 3234 3338 3835  acecode:.0243885
    0x01a0:  3134 3030 3534 3733 3330 3331 3430 3131  1400547330314011
    0x01b0:  3632 310d 0a54 7261 6e73 6665 722d 456e  621..Transfer-En
    0x01c0:  636f 6469 6e67 3a20 6368 756e 6b65 640d  coding:.chunked.
    0x01d0:  0a0d 0a66 640d 0a7b 2274 696d 6573 7461  ...fd..{"timesta
    0x01e0:  6d70 223a 3135 3437 3634 3338 3433 3831  mp":154764384381
    0x01f0:  312c 2273 7461 7475 7322 3a34 3135 2c22  1,"status":415,"
    0x0200:  6572 726f 7222 3a22 556e 7375 7070 6f72  error":"Unsuppor
    0x0210:  7465 6420 4d65 6469 6120 5479 7065 222c  ted.Media.Type",
    0x0220:  2265 7863 6570 7469 6f6e 223a 226f 7267  "exception":"org
    0x0230:  2e73 7072 696e 6766 7261 6d65 776f 726b  .springframework
    0x0240:  2e77 6562 2e48 7474 704d 6564 6961 5479  .web.HttpMediaTy
    0x0250:  7065 4e6f 7453 7570 706f 7274 6564 4578  peNotSupportedEx
    0x0260:  6365 7074 696f 6e22 2c22 6d65 7373 6167  ception","messag
    0x0270:  6522 3a22 436f 6e74 656e 7420 7479 7065  e":"Content.type
    0x0280:  2027 6170 706c 6963 6174 696f 6e2f 782d  .'application/x-
    0x0290:  7777 772d 666f 726d 2d75 726c 656e 636f  www-form-urlenco
    0x02a0:  6465 643b 6368 6172 7365 743d 5554 462d  ded;charset=UTF-
    0x02b0:  3827 206e 6f74 2073 7570 706f 7274 6564  8'.not.supported
    0x02c0:  222c 2270 6174 6822 3a22 2f76 312f 6a6f  ","path":"/v1/jo
    0x02d0:  6273 227d 0d0a                           bs"}..
{% endhighlight %} 

前面的协议头这里省略了，我们看返回的body体，两个换行符+fd+换行符+json体，按照我的理解，json解析明显会存在问题，因为增加了 fd + 换行符， 而实际上并没有出现这种状况，仔细阅读了http协议，发现还是自己傻缺了，<a href="https://tools.ietf.org/html/rfc7230" target="_blank">http1.1协议</a>, 简单说还是对http协议理解不够充分

## 二. http1.1协议的Transfer-Encoding

1. Transfer-Encoding 是http1.1新增的，所以要求接收http请求的服务器具备http1.1的解析能力；如果http
