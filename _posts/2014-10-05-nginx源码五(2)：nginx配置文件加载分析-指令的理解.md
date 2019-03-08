---
layout: content
title: nginx源码五(2)：nginx配置文件分析-指令的理解
status: 3
complete: 10% 
category: nginx
---

## 一.前言

前面对`location {}`的配置分析起了一个头，实际上，location的指令非常复杂，关注一下下面几个问题：
- 1、nginx通过指令和指令的handle方式处理 `http`, `server`, `location`的层级关系，那么下面那几种层级关系的配置哪些是可以支持的呢？

![ngx_location_conf](/images/nginx/ngx_cmd1.jpg)

- 2、nginx在指令中如何区分`http {}` 和 `error_page` 这两种不同的指令类型？

这是本篇文章探讨的内容，先讨论nginx的指令嵌套问题

## 二.nginx指令的嵌套问题

我们先看一下nginx的`command`数据结构，我在后面做了注释
{% highlight c %}
struct ngx_command_s {
    ngx_str_t             name; // 指令名称
    ngx_uint_t            type;  // 指令类型
    char               *(*set)(ngx_conf_t *cf, ngx_command_t *cmd, void *conf); // 设置方法
    ngx_uint_t            conf;    // 上下文节点中指向配置的选择
    ngx_uint_t            offset;  // 配置内存储改元素信息的偏移量
    void                 *post;    // 通常是NULL
};
{% endhighlight %}

在这里面，`ngx_command_s` 的 `type` 成员变量是最复杂的，nginx在指令分析中，会根据type的类型做相应的处理，`type`有各种类型，主要在这几个方面
- 1、参数方面，设定一个指令接受参数的个数
- 2、指令方面
    - 指令的类型，是否是支持嵌套形式的指令
    - 指令接收参数的要求，参数要求的个数，一个参数还是多个参数，是否支持bool等等
- 3、指令类型在配置参数存储位置的影响

下面是一张图，标识nginx的command如何通过nginx_uint_t类型来标识不同的类型，一个nginx_uint_t类型占四个字节，nginx采用各个字节来存储不同的类型。同时可以通过 或的方式来标识各个类型的集合

![ngx_conf](/images/nginx/ngx_cmd2.jpg)


**下面是对各个类型的一个注释**
{% highlight c %}
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
{% endhighlight %}


![ngx_conf](/images/nginx/ngx_cmd3.jpg)


![ngx_conf](/images/nginx/ngx_cmd4.jpg)











{% highlight c %}

static ngx_int_t
ngx_conf_handler(ngx_conf_t *cf, ngx_int_t last)
{
    ...
    for (i = 0; ngx_modules[i]; i++) {
        cmd = ngx_modules[i]->commands;
        ...
        for ( /* void */ ; cmd->name.len; cmd++) {
            ...
            // 上下文的类型是否一致
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
{% endhighlight %}



{% highlight c %}
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
{% endhighlight %}


## 三.nginx的location查找算法

接着前面对配置文件的分析，我们知道一个server{}里面可以设定多个location {}， 那么请求来了，怎么找到对应的location {} 处理呢，依次去对比，如果location {} 配置的太多，性能会存在问题。

在这里nginx采用了三叉树的方法，三叉树的特点是这样的：

1. 比字典树省空间
2. 搜索效率比较高

nginx在location匹配的时候，还遵循下面的流程

精确匹配  ===> 前缀匹配 ===> 正则匹配

这一切是怎么实现的呢？ 这是本章的重点：

