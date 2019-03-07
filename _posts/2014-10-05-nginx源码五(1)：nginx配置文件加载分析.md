---
layout: content
title: nginx源码五(1)：nginx配置加载分析
status: 1
complete: 100% 
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
    conf.module_type = NGX_CORE_MODULE;
    conf.cmd_type = NGX_MAIN_CONF;

#if 0
    log->log_level = NGX_LOG_DEBUG_ALL;
#endif

    if (ngx_conf_param(&conf) != NGX_CONF_OK) {
        environ = senv;
        ngx_destroy_cycle_pools(&conf);
        return NULL;
    }

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

`创建配置项；分析配置文件；初始化未定义的默认值`，这三件事情是配置处理的主思路，但是针对配置分析的流程，说的还是不细，我们下面详细说配置分析的流程，并且说一下配置的最终存储结构，我们拿到一个简单的配置文件，说一下nginx的处理思路

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

主要步骤：
- `1. 首先Nginx给每个模块都做了编号，编号信息存在插件的index变量里面`
- `2. 申请一段连续的内存，准备存放各个插件的上下文信息的访问指针`
- `3. 调起所有的NGX_CORE_MODULE插件，初始化插件的配置`
- `4. 开始分析配置文件指令，首先分析下面的指令类别`
    - conf.module_type = NGX_CORE_MODULE;
    - conf.cmd_type = NGX_MAIN_CONF;
- `5. 分析到第一个指令"worker_processes"， 遍历查找所有 NGX_CORE_MODULE 插件，指令类型是 NGX_MAIN_CONF 的指令，执行指令对应的handle`
- `6. 对于"worker_processes"， 因为前面已经创建配置信息，配置信息获取方式如下图，所以这里可以直接更新到ngx_core_conf_t类型的结构体里`
![ngx_http_conf](/images/nginx/ngx_conf.jpg)

- `7. 配置文件继续分析，分析到event指令 和 "{"`
- `8. 调起event指令的处理handle "ngx_events_block"`

{% highlight c %}
static ngx_command_t  ngx_events_commands[] = {

    { ngx_string("events"),
      NGX_MAIN_CONF|NGX_CONF_BLOCK|NGX_CONF_NOARGS,
      ngx_events_block,
      0,
      0,
      NULL },

      ngx_null_command
};
{% endhighlight %}

- `9. ngx_events_block 开启event模块的处理流程，首先创建连续的内存，存放所有NGX_EVENT_MODULE类型模块上下文指针，这段连续内存的访问方式见上图标识`
- `10. 启动分析block内的指令`
    - cf->module_type = NGX_EVENT_MODULE;
    - cf->cmd_type = NGX_EVENT_CONF;
- `11. 分析完配置后，写到第9步NGX_EVENT_MODULE类型模块上下文指针指向的上下文配置信息里面`

<br/>
**这里面能够看出一个思路：配置分类型，实际也是分了权限，管理方式是，NGX_CORE_MODULE类型模块管理NGX_EVENT_MODULE类型模块，而下面要分析的是http指令，http的配置情况有所不同，我们看一下下面的图，实际存在下面几个特点**
<br/>

- http，server，location之间关系是一个多叉树结构
![ngx_http_conf](/images/nginx/ngx_conf2.jpg)

- 配置的选择顺序按照 location 到 server 到 http先后关系，例如 root这个指令，它可以在 http， server， location三个级别做配置；那么对于location匹配到的请求，选择配置信息的顺序是，如果 location {} 里面配置了，选用location{}，否则就是 server {}, 在次是http {}, 如果都么有配置，那么必定有一个默认值
{% highlight c %}
    { ngx_string("root"),
      NGX_HTTP_MAIN_CONF|NGX_HTTP_SRV_CONF|NGX_HTTP_LOC_CONF|NGX_HTTP_LIF_CONF
                        |NGX_CONF_TAKE1,
      ngx_http_core_root,
      NGX_HTTP_LOC_CONF_OFFSET,
      0,
      NULL },
{% endhighlight %}

- 理论上，nginx可以配置无限个server，每个server又可以配置无限多个location。那么我们考虑一个请求过来，先要找到对应的server，在找到对应的location，根据location的配置确定处理逻辑；配置信息的存储实际第一步是按照下图的关系做了原始存储，那么存储的流程是什么样的呢？  我们接着上面的配置文件做分析

![ngx_http_conf](/images/nginx/ngx_http_conf.jpg)

- `12. nginx分析到http指令，执行http的指令 ngx_http_block`
{% highlight c %}
static ngx_command_t  ngx_http_commands[] = {

    { ngx_string("http"),
      NGX_MAIN_CONF|NGX_CONF_BLOCK|NGX_CONF_NOARGS,
      ngx_http_block,
      0,
      0,
      NULL },

      ngx_null_command
};
{% endhighlight %}

- `13. ngx_http_block在处理的时候，考虑到内部还有server block 和 location block的特点，做了下面一个设计`
    - 设计一个上下文的节点数据结构，包含三个成员指针，main_conf、srv_conf、loc_conf
    - 上下文节点的三个成员指针，指向的是三个 NGX_HTTP_MODULE 的插件的配置指针数组，如上图图示

- `14. 继续http的block内的配置分析`
    - cf->module_type = NGX_HTTP_MODULE;
    - cf->cmd_type = NGX_HTTP_MAIN_CONF;

- `15. 分析到 default_type 指令，和前面的流程一样，找到 NGX_HTTP_MODULE 类型下的对应的插件 ngx_http_core_module`
- `16. 执行ngx_http_core_module的 default_type 指令， set用户配置`
- `17. 继续server的block的处理和location的block处理，流程是一样的`

<br/>
**这里有一个非常关键的问题，我们在后面写nginx的模块的时候需要关注的，关于写一个插件的时候指令的设定要求，从上面那张图可以看出，server{} 内 main_conf 和 http{} 的main_conf是共用的，location{}里面的main_conf，srv_conf和 server{} 里面的main_conf 和 srv_conf是共用的，那么需要关注：**

1. 在配置一个指令的时候，如果配置了http{}，server{}，location{}三处都可以设定指令，那么这个指令配置的信息应该放在loc_conf指向的配置数组中
2. 在配置一个指令的时候，如果配置了http{}，server{} 两处可以设定指令，那么这个指令配置的信息应该放在srv_conf指向的配置数组中

**从下面的代码可以看出来，这两个规范，代码中实际没有严格要求，但是如果写nginx插件的时候，没有按照这个要求，会导致location{}内配置命令的相互干扰**

{% highlight c %}
{ ngx_string("connection_pool_size"),
      NGX_HTTP_MAIN_CONF|NGX_HTTP_SRV_CONF|NGX_CONF_TAKE1, //支持 http{}，server{}内设定， 支持一个参数
      ngx_conf_set_size_slot,  // set方法
      NGX_HTTP_SRV_CONF_OFFSET,    // 存在 srv_conf
      offsetof(ngx_http_core_srv_conf_t, connection_pool_size), //配置数据结构里面的存放位置, 通过偏移量方式
      &ngx_http_core_pool_size_p },

{ ngx_string("types_hash_max_size"),
      NGX_HTTP_MAIN_CONF|NGX_HTTP_SRV_CONF|NGX_HTTP_LOC_CONF|NGX_CONF_TAKE1, //支持 http{}，server{}，location{}内设定
      ngx_conf_set_num_slot,   
      NGX_HTTP_LOC_CONF_OFFSET,   // 存在 loc_conf
      offsetof(ngx_http_core_loc_conf_t, types_hash_max_size), 
      NULL },

{% endhighlight %}

下面，我们在画一张图来标识用户指令配置的存储位置

![ngx_http_conf](/images/nginx/ngx_conf3.jpg)

到这一步，我们已经知道 解析完 http{} 这个block后，存储到配置中；图中也标识出来，通过`ngx_http_core_module`对应的配置管理多个server的conf，那么下一步需要做的是
1. 解决如何通过host快速找到server以及server的所有配置信息
2. 找到server后，如何快速找到location

**由于配置信息分散在各个上下文节点指向的配置数组中，所以，我们要做的第一步是配置信息的merge，下图是`配置信息merge的过程`，红框就是merge好，取各个配置的地方。**

![ngx_http_conf](/images/nginx/ngx_conf4.jpg)

一个server对应理论上无限个location，我们看到前面的图示，location信息通过queue来串连，在接收到request的时候，如果依次去匹配查找location，性能肯定会存在问题，对此，nginx做了一个处理
- 对于精确匹配和前缀匹配，构建一个三叉查找树
- 如果精确匹配和前缀匹配都找不到location，会通过正则方式去查找(支持pcre库的前提下)

构造三叉树和实现查找这里不在详细描述，放在后面一篇文章继续

## 四. 总结

本篇文章简单介绍了nginx配置文件的分析和配置信息存储的过程，在配置分析的过程中，还有三部分主要的内容
- 构造location的静态树(三叉查找树) (ngx_http_init_static_location_trees)
- nginx的request处理过程的状态机设计 (ngx_http_init_phase_handlers)
- 将事件和处理的server关联的过程 (ngx_http_optimize_servers)

在后面将继续分析这三部分内容


