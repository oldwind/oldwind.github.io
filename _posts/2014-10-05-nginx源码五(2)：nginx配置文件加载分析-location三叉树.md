---
layout: content
title: nginx源码五(2)：nginx配置文件分析-location三叉树
status: 3
complete: 10% 
category: nginx
---

## 一.前言

前面对`location {}`的配置分析起了一个头，实际上，location的指令非常复杂，体现在几个方面：
1. nginx通过指令和指令的handle方式处理 `http`, `server`, `location`的层级关系，那么下面那几种层级关系的配置哪些是可以支持的呢？

![ngx_location_conf](/images/nginx/ngx_location1.jpg)

2. 请求来到后，需要找到server配置，找到server配置后，要找到对应的location配置，理论上loation配置支持无数个，从前面的代码可以看出来


接着前面对配置文件的分析，我们知道一个server{}里面可以设定多个location {}， 那么请求来了，怎么找到对应的location {} 处理呢，依次去对比，如果location {} 配置的太多，性能会存在问题。

在这里nginx采用了三叉树的方法，三叉树的特点是这样的：

1. 比字典树省空间
2. 搜索效率比较高

nginx在location匹配的时候，还遵循下面的流程

精确匹配  ===> 前缀匹配 ===> 正则匹配

这一切是怎么实现的呢？ 这是本章的重点：


## 二.配置

{% highlight bash %}
http{
    server {
        listen       8081;
        server_name  localhost somename  alias  another.alias;
        location / {
            add_header Content-Type "text/html";
            return 200  "/";
        }

        # 嵌套匹配
        location /ab {
            add_header Content-Type "text/html";
            return 200 "/ab";

            location /abc {
                add_header Content-Type "text/html";
                return 200 "/abc";

                 location /abcd {
                    add_header Content-Type "text/html";
                    return 200 "/abc";
                }
            }
        }
    }
}
{% endhighlight %}
