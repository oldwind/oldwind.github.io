---
layout: content
title: nginx源码五(3)：nginx配置文件分析-location三叉树
status: 3
complete: 10% 
category: nginx
---

## 一.前言

我们在前面提到了nginx的指令分析，针对server {} 里面的 location {} 指令，平行的 location指令 和 嵌套的 location指令 之间都会通过 queue 的双向链表来进行串联， 如下图：

![ngx_loc_tree_conf](/images/nginx/ngx_loctree1.jpeg)

在建立双向链表关系后，下一步，nginx 在 `ngx_http_init_locations` 方法中做了 排序 和 queue 的切分的工作， 我们在这里选择构造一个例子来表述一下queue的情况

{% highlight c %}
http {
    server {
        add_header  Content-Type 'text/html; charset=utf-8';
        location / {
           return 200 "0";
        }

        location = / {
            return 200 "1";
        }

        location  /a {
            return 200 "2";
        }

        location = /a {
            return 200 "3";
        }

        location = /ab {
            return 200 "4";
        }

        location /ab {
            return 200 "5";
        }

        location @my {
            return 200 "6";
        }

        location ~ /abc* {
            return 200 "7";
        }
    }
}
{% endhighlight %}


exact(sorted) -> inclusive(sorted) -> regex -> named -> noname

排序规则如下：
1. 精确匹配 > 前缀匹配 > 正则匹配 > named类型匹配 > noname类型匹配



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

