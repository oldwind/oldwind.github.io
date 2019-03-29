---
layout: content
title: nginx源码十五：nginx的body体控制
status: 2 
category: nginx
---

配置nginx参数

failed to open stream: HTTP request failed! HTTP/1.1 413 Request Entity Too Large


client_max_body_size 100; 控制大小

多个包发送给服务端，服务端接收成功后