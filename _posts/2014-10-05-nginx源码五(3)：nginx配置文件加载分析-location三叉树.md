---
layout: content
title: nginx源码五(3)：nginx配置文件分析-location三叉树
status: 3
complete: 10% 
category: nginx
---

## 一.前言

我们在前面提到了nginx的指令分析，针对server {} 里面的 location {} 指令，平行的 location指令 和 嵌套的 location指令 之间都会通过 queue 的双向链表来进行串联， 双向链表的头节点分别是 location {} 的父 location指向的 ngx_http_core_moudle的配置文件区，如下图

![ngx_loc_tree_conf](/images/nginx/ngx_loc_data.jpeg)

上面的图只是location {} 指令的分析的初始化阶段，下一步，会经历多个步骤，最终构建一个location的静态树，方便nginx在处理http请求的时候，在找到server后，快速的找到location的配置，总的来说，分成下面几个步骤：
- 1.location配置的创建
- 2.location指令的排序
- 3.location指令的切分
- 4.location精确匹配和包含匹配的合并
- 5.处理location的包含关系构成list
- 6.location的三叉树形成

另外我们也去分析一下对于request请求的uri匹配一下


为了便于理解，我在这里设计了一个location的示例，我们看一下三叉树的形成过程
 
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

        location = /ab {
            return 200 "2";
        }

        location /ba {
            return 200 "6";
        }

        location /bab {
            return 200 "7";
        }

        location /ab {
            location /abc {
                return 200 "3.1";
            }
            return 200 "3";
        }

        location /abd {
            return 200 "4";
        }

        location /abe {
            return 200 "5";
        }

        location @my {
            return 200 "8";
        }

        location ^~ /c {
            return 200 "9";
        }

        location ~ /cab[0-9]* {
            return 200 "10";
        }

        location /d {
            error_page 404 = @my;
        }

        location /e {         
            limit_except GET POST {
                deny  all;
            }
        }
    }
}
{% endhighlight %}


## 二.location配置的创建

server{} 指令内部的location {} 指令，是一对多的关系，同时location {}内部还支持嵌套，nginx在分析完每一条指令后，会将配置信息写到创建的配置管理中，我们在开始的图中，描述了这个配置的关系，在配置信息创建后，下面就可以处理这些配置，走到下面第三步，对指令配置做一个排序。 为了方便理解，我们按照上面的 location的配置示例，画了下面一张图， 在这里，简单起见，我们不考虑 location内部的嵌套关系

![ngx_loc_tree_conf](/images/nginx/ngx_loc_init.jpeg)


## 三.location指令的排序

nginx通过queue去串联配置的location，下一步是对location进行排序，排序算法的实现，在前面关于ngx_queue_t的数据结构分析中，做了解释，简单说就是：
1. ngx_queue_t实现自己的排序逻辑
2. 使用的业务实现自己的比较逻辑

nginx的排序逻辑采用的是插入排序的方案，

![ngx_loc_tree_conf](/images/nginx/ngx_loc_split1.jpeg)


## 四.location指令的切分

![ngx_loc_tree_conf](/images/nginx/ngx_loc_split2.jpeg)

## 五.location指令的精确匹配和包含匹配的合并

![ngx_loc_tree_conf](/images/nginx/ngx_loc_exac.jpeg)

## 六.location指令生成包含关系的list

![ngx_loc_tree_conf](/images/nginx/ngx_loc_list.jpeg)

## 七.location的三叉树形成

![ngx_loc_tree_conf](/images/nginx/ngx_loc_tree.jpeg)

## 八. 查找算法 ngx_http_core_find_static_location


## 九.总结

一个请求发起到location {}，中间还要经历一层rewrite过程，rewrite是通过nginx的模块化来实现，在ngx_http_rewrite_module.c 中做了实现，我们在这里后面 <a href="/index.html#/nginx/2014/10/14/nginx源码十四(1)-nginx的rewrite模块分析.html" title="nginx源码十四(1)：nginx的rewrite模块分析">nginx源码十四(1)：nginx的rewrite模块分析</a> 在接着继续分析