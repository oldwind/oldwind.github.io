---
layout: content
title: ssh隧道链接远程docker
status: 1 
category: java
author:   "yimuren"
tags:
    - docker
---

## 一. 前言

很简单的一个问题，这里整理一下

## 二.启动docker

指定转发端口

```bash
/usr/bin/dockerd -H tcp://localhost:2375 -H unix:///var/run/docker.sock
```

## 三. 建立ssh隧道

```bash
export DOCKER_HOST=tcp://localhost:2376
ssh -L 2376:localhost:2375 username@src_ip
```
