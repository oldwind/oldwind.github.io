---
layout: content
title: nginx源码五(2)：nginx配置文件分析-指令的理解
status: 1
complete: 10% 
author:     "yimuren"
tags:
    - nginx
---

## 一.前言

前面对`location {}`的配置分析起了一个头，实际上，location的指令非常复杂，关注一下下面几个问题：
- 1、nginx通过指令和指令的handle方式处理 `http`, `server`, `location`的层级关系，那么下面那几种层级关系的配置哪些是可以支持的呢？

![ngx_location_conf]({{site.baseurl}}/img/nginx/ngx_cmd1.jpg)

- 2、nginx在指令中如何区分`http {}` 和 `error_page` 这两种不同的指令类型？

这是本篇文章探讨的内容，先讨论nginx的指令嵌套问题

## 二.nginx指令的嵌套问题

我们先看一下nginx的`command`数据结构，我在后面做了注释
```c
struct ngx_command_s {
    ngx_str_t             name; // 指令名称
    ngx_uint_t            type;  // 指令类型
    char               *(*set)(ngx_conf_t *cf, ngx_command_t *cmd, void *conf); // 设置方法
    ngx_uint_t            conf;    // 上下文节点中指向配置的选择
    ngx_uint_t            offset;  // 配置内存储改元素信息的偏移量
    void                 *post;    // 通常是NULL
};
```

在这里面，`ngx_command_s` 的 `type` 成员变量是最复杂的，nginx在指令分析中，会根据type的类型做相应的处理，`type`有各种类型，主要在这几个方面
- 1、参数方面，设定一个指令接受参数的个数
- 2、指令方面
    - 指令的类型，是否是支持嵌套形式的指令
    - 指令接收参数的要求，参数要求的个数，一个参数还是多个参数，是否支持bool等等
- 3、指令类型在配置参数存储位置的影响

下面是一张图，标识nginx的command如何通过nginx_uint_t类型来标识不同的类型，一个nginx_uint_t类型占四个字节，nginx采用各个字节来存储不同的类型。同时可以通过 或的方式来标识各个类型的集合

![ngx_conf]({{site.baseurl}}/img/nginx/ngx_cmd2.jpg)


**下面是对各个类型的一个注释**
```c
#define NGX_CONF_NOARGS      0x00000001    // 没有参数
#define NGX_CONF_TAKE1       0x00000002    // 1个参数
#define NGX_CONF_TAKE2       0x00000004    // 2个参数
#define NGX_CONF_TAKE3       0x00000008    // 3个参数
#define NGX_CONF_TAKE4       0x00000010    // 4个参数
#define NGX_CONF_TAKE5       0x00000020    // 5个参数
#define NGX_CONF_TAKE6       0x00000040    // 6个参数
#define NGX_CONF_TAKE7       0x00000080    // 7个参数

#define NGX_CONF_BLOCK       0x00000100    // 指令支持嵌入
#define NGX_CONF_FLAG        0x00000200    // 指令支持 on, off
#define NGX_CONF_ANY         0x00000400    // 指令
#define NGX_CONF_1MORE       0x00000800    // 指令最少要1个参数
#define NGX_CONF_2MORE       0x00001000    // 指令最少要2个参数
#define NGX_CONF_MULTI       0x00000000     /* compatibility */

#define NGX_DIRECT_CONF      0x00010000    // 
#define NGX_MAIN_CONF        0x01000000    // 
#define NGX_ANY_CONF         0x0F000000    // 
```

`NGX_DIRECT_CONF`、`NGX_MAIN_CONF`、`NGX_ANY_CONF`的含义不是非常好描述，我们拿到几个代表性的配置块来看一下，参考下图，分别是
- core模块
- http {} 的分析
- event {} 的分析

![ngx_conf]({{site.baseurl}}/img/nginx/ngx_cmd3.jpg)

`NGX_DIRECT_CONF`、`NGX_MAIN_CONF`、`NGX_ANY_CONF`影响的是用户配置信息的设定，这里的思考大概是这样的：
- 1、对于`NGX_DIRECT_CONF`存在配置中全局作用域的command，例如 `daemon` `master_process` 这类型的指令，分析到这类型的指令，拿着上下文的节点，可以直接找到指令所需要配置的位置去设定
- 2、`NGX_MAIN_CONF` 可以用于支持一级上下文节点的存储，可以参考下面图，对于分析到 http {}， 那么http的上下文节点指针要存在上一级指针数组中
- 3、其它的配置，基本思路是针对不同上下文节点，通过偏移拿到配置的头节点
可以参考下图

![ngx_conf]({{site.baseurl}}/img/nginx/ngx_cmd4.jpg)

具体可以参考下面的代码实现

```c
static ngx_int_t
ngx_conf_handler(ngx_conf_t *cf, ngx_int_t last)
{
    ...
            if (cmd->type & NGX_DIRECT_CONF) {
                conf = ((void **) cf->ctx)[ngx_modules[i]->index];

            } else if (cmd->type & NGX_MAIN_CONF) {
                conf = &(((void **) cf->ctx)[ngx_modules[i]->index]);

            } else if (cf->ctx) {
                confp = *(void **) ((char *) cf->ctx + cmd->conf);

                if (confp) {
                    conf = confp[ngx_modules[i]->ctx_index];
                }
            }
    ...
}
```


## 三. 提出问题的回答

我们回答最开始的问题，解答一下，`location是可以无限嵌套的，但是有些条件`，nginx在实现这个功能是非常有意思的，为什么这样做，实现的场景是啥，还不是非常理解

但是在实现上，http{} 作用域内哪些指令会被执行，nginx通过 上面说的 cmd 的 cmd_type 来管理， 我们来看 server指令和 location指令，可以发现location指令在做conf的parse时候，会支持 NGX_HTTP_LOC_CONF的模块，这是location能嵌套的原因

```c
    cf->module_type = NGX_HTTP_MODULE;
    cf->cmd_type = NGX_HTTP_MAIN_CONF;
    rv = ngx_conf_parse(cf, NULL);
```


```c
    { ngx_string("server"),
      NGX_HTTP_MAIN_CONF|NGX_CONF_BLOCK|NGX_CONF_NOARGS,
      ngx_http_core_server,
      0,
      0,
      NULL },
```

```c
    { ngx_string("location"),
      NGX_HTTP_SRV_CONF|NGX_HTTP_LOC_CONF|NGX_CONF_BLOCK|NGX_CONF_TAKE12,
      ngx_http_core_location,
      NGX_HTTP_SRV_CONF_OFFSET,
      0,
      NULL },
```


```c
static ngx_int_t
ngx_conf_handler(ngx_conf_t *cf, ngx_int_t last)
{
    ...
    found = 0;
    for (i = 0; ngx_modules[i]; i++) {
        cmd = ngx_modules[i]->commands;
        ...
        for ( /* void */ ; cmd->name.len; cmd++) {
            ...
            // 指令类型和
            if (!(cmd->type & cf->cmd_type)) {
                continue;
            }
            ...
        }
    }

    if (found) {
        ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                           "\"%s\" directive is not allowed here", name->data);
        return NGX_ERROR;
    }
    ...
}
```

**nginx 规定了 location 的嵌套条件，可参考下面的代码**
- `如果父location 指令是 是精确匹配，locaton不能嵌套`
- `如果父location 指令是 @名称这种，不能发生嵌套匹配`
- `如果是 @名称这种，不能发生嵌套匹配`
- `嵌套匹配时候，和父location的参数，保持包含关系，例如 父location 是 /a， 子location 必须是 /a* 这种`

```c
        if (pclcf->exact_match) {
            ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                               "location \"%V\" cannot be inside "
                               "the exact location \"%V\"",
                               &clcf->name, &pclcf->name);
            return NGX_CONF_ERROR;
        }

        if (pclcf->named) {
            ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                               "location \"%V\" cannot be inside "
                               "the named location \"%V\"",
                               &clcf->name, &pclcf->name);
            return NGX_CONF_ERROR;
        }

        if (clcf->named) {
            ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                               "named location \"%V\" can be "
                               "on the server level only",
                               &clcf->name);
            return NGX_CONF_ERROR;
        }

```
