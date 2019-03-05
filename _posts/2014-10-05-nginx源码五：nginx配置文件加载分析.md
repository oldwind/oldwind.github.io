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
- `键值对` 
.etc

不同的格式可以有不同的lib库处理，nginx对外部库的依赖非常的少，对配置文件的分析也是自己做的，nginx配置文件的格式是一个综合体，看起来类似键值对，但是，也支持一键 对 多个参数的方式。 同时nginx的配置存在多个层级关系，例如command里面的

{% highlight bash%}
    event{
    }
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

我们从两点来说nginx的配置文件分析过程，第一点，流程；如同定义一个变量，从声明到使用，有一个流程，在程序里面，我们可以先声明，在赋值，然后使用； 也可以声明和赋值一起，然后使用；nginx在处理配置的分析时候，严格定义了流程

### 2.1 conf的create
所谓的create；就是对所有模块设定的conf的数据结构的成员变量赋默认值，例如下面的`ngx_core_module_create_conf`
   
{% highlight c%}
static ngx_core_module_t  ngx_core_module_ctx = {
    ngx_string("core"),
    ngx_core_module_create_conf,
    ngx_core_module_init_conf
};


ngx_module_t  ngx_core_module = {
    NGX_MODULE_V1,
    &ngx_core_module_ctx,                  /* module context */
    ngx_http_core_commands,                /* module directives */
    ...
};

static void *ngx_core_module_create_conf(ngx_cycle_t *cycle) {
    ngx_core_conf_t  *ccf;

    ccf = ngx_pcalloc(cycle->pool, sizeof(ngx_core_conf_t));
    if (ccf == NULL) {
        return NULL;
    }
    ...

}
{% endhighlight %}

### 2.2 配置文件解析

将分析结果写到对应配置的conf数据结构里面

{% highlight c%}

if (ngx_conf_parse(&conf, &cycle->conf_file) != NGX_CONF_OK) {
    environ = senv;
    ngx_destroy_cycle_pools(&conf);
    return NULL;
}
{% endhighlight %}

### 2.3 配置信息的默认值
如果有些配置信息没有配置，则赋予默认值

{% highlight c%}

static char *
ngx_core_module_init_conf(ngx_cycle_t *cycle, void *conf)
{
    ngx_core_conf_t  *ccf = conf;

    ngx_conf_init_value(ccf->daemon, 1);
    ngx_conf_init_value(ccf->master, 1);
    ...
}
{% endhighlight %}

## 三.nginx配置存储结构

`创建配置项，分析配置文件，初始化未定义的默认值`，这三件事情是配置处理的主思路，但是针对配置分析的流程，说的还是不细，我们下面详细说配置分析的流程，并且说一下配置的最终存储结构， 我们拿到一个简单的配置文件，说一下nginx的处理思路

{% highlight bash%}
worker_processes  1;

events {
    worker_connections  1024;
    use epoll;
}


http {
    default_type  application/octet-stream;

    server {
        listen       80;
        server_name  localhost;

        location / {
            root   html;
            index  index.html index.htm;
        }
    }
}

keepalive_timeout  65;
{% endhighlight %}

1. 首先Nginx给每个模块都做了编号，现在要存储每个模块的上下文信息，通过模块编号来存取





前面已经分析到，nginx拥有插件化(module)的架构，每个插件需要关心的问题是在不同阶段的静态信息获取，例如用户在配置阶段对插件的指令做了配置，那么在执行阶段如何获取到这个指令的配置信息？ 

另外，nginx的配置信息具备一定的"层次性"，分层的方法是通过 "{}"来实现，例如 "event {}"，在这里，`event`指令是通过`ngx_event_module`来管理，


另外，指令的配置并不是一个一维数组，存在下面几种情况
- NGX_CORE_MODULE 的指令配置； 
    - 我们可以看下面一张图，对于一个核心模块，例如`ngx_core_module`，他的指令是唯一的，像`worker_processes`指令
- NGX_EVENT_MODULE 的指令配置；
    - 指令有了分层的概念，`event {}`， 这里面`event`指令是
- NGX_HTTP_MODULE 的指令配置

其它的几种module的管理方式，和这三种类别一致，这里不一一详细描述了


配置信息的存储结构需要考虑一个问题，




![ngx_http_conf](/images/nginx/ngx_conf.jpg)


![ngx_http_conf](/images/nginx/ngx_conf2.jpg)


![ngx_http_conf](/images/nginx/ngx_http_conf.jpg)

