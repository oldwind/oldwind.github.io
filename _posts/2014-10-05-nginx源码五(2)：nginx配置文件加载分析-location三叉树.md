---
layout: content
title: nginx源码五(2)：nginx配置文件分析-location三叉树
status: 3
complete: 10% 
category: nginx
---

## 一.前言

前面对`location {}`的配置分析起了一个头，实际上，location的指令非常复杂，体现在几个方面：
- 1、nginx通过指令和指令的handle方式处理 `http`, `server`, `location`的层级关系，那么下面那几种层级关系的配置哪些是可以支持的呢？

![ngx_location_conf](/images/nginx/ngx_location1.jpg)

- 2、请求来到后，需要找到server配置，找到server配置后，要找到对应的location配置，理论上loation配置支持无数个，因为从前面的代码可以看出来，分析到location指令后，是独立的申请内存，不同location是通过`ngx_queue_t`来串联起来的，并没有限制个数，那么在这个时候，location的匹配规则是什么？处理的流程是什么样的

这些都是本篇文章探讨的内容，先讨论nginx的指令嵌套问题

## 二.nginx指令的嵌套问题

nginx的指令嵌套重点关注两部分代码，一是

{% highlight c %}
struct ngx_command_s {
    ngx_str_t             name;
    ngx_uint_t            type;
    char               *(*set)(ngx_conf_t *cf, ngx_command_t *cmd, void *conf);
    ngx_uint_t            conf;
    ngx_uint_t            offset;
    void                 *post;
};
{% endhighlight %}




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

## 三.nginx的location查找算法

接着前面对配置文件的分析，我们知道一个server{}里面可以设定多个location {}， 那么请求来了，怎么找到对应的location {} 处理呢，依次去对比，如果location {} 配置的太多，性能会存在问题。

在这里nginx采用了三叉树的方法，三叉树的特点是这样的：

1. 比字典树省空间
2. 搜索效率比较高

nginx在location匹配的时候，还遵循下面的流程

精确匹配  ===> 前缀匹配 ===> 正则匹配

这一切是怎么实现的呢？ 这是本章的重点：

