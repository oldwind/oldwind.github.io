---
layout: content
title: nginx源码五：nginx配置加载分析
status: 3
complete: 20% 
category: nginx
---

## 一.前言
应用系统在启动前要处理一些用户定制化的需求，这个就是配置，配置有两种方式，一种是二进制文件，这种目前已经很少了；另外一种是text文本，配置的数据格式有多种，一般有
- `JSON`
- `YAML`
- `XML`
- `键值对` .etc

不同的格式可以有不同的lib库处理，nginx对外部库的依赖非常的少，对配置文件的分析也是自己做的，nginx配置文件的格式是一个综合体，看起来类似键值对，但是，也支持一键 对 多个参数的方式。 同时nginx的配置存在多个层级关系，例如command里面的

{% highlight bash%}
    http {
        server {
            location {

            }
        }
    }
{% endhighlight%}

配置文件中的层级关系nginx采用了command + {} 的方式，nginx的配置文件类似一门简单的语言，nginx作为内核，分析了配置文件，将配置参数记录下来，提供程序运行时使用。

我们这次的文章不会详细介绍nginx如何分析配置配置文件，分析的过程非常复杂，我们重点讲一下，
- nginx配置文件分析的流程
- nginx配置信息的存储配置数据结构
- nginx配置信息的获取方式


## 二.nginx的配置文件的分析流程






## 三.nginx配置存储结构

