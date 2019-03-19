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

上面的图只是location {} 指令的分析的初始化阶段，下一步，会经历多个步骤，从思路上，可以归纳成
- location指令的初始化
- location指令的排序
- location指令的切分
- location精确匹配和包含匹配的合并
- 处理location的包含关系构成list
- location指令构成三叉数结构

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


## 二.location指令的初始化

首先说明的是，在location指令初始化中，我们先不考虑嵌套的情况，嵌套的处理
![ngx_loc_tree_conf](/images/nginx/ngx_loc_init.jpeg)

## 三.location指令的排序

![ngx_loc_tree_conf](/images/nginx/ngx_loc_split1.jpeg)


## 四.location指令的切分

![ngx_loc_tree_conf](/images/nginx/ngx_loc_split2.jpeg)

## 五.location指令的精确匹配和包含匹配的合并

![ngx_loc_tree_conf](/images/nginx/ngx_loc_exac.jpeg)

## 六.location指令生成包含关系的list

![ngx_loc_tree_conf](/images/nginx/ngx_loc_list.jpeg)

## 七.location指令构成三叉数结构

![ngx_loc_tree_conf](/images/nginx/ngx_loc_tree.jpeg)

## 八. 查找算法 ngx_http_core_find_static_location

