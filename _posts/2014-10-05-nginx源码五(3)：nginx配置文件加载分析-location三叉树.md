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

在建立双向链表关系后，下一步，nginx 会做一些什么操作呢？ 我们拿一个location 的事例来说一下
 
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

        location /ba {
            return 200 "6";
        }

        location /bab {
            return 200 "7";
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
            error_page 404 = @my;
        }
    }
}
{% endhighlight %}

第一步的情况我们在前面已经说了，创建一个双向链表(ngx_queue_t)

## 二.nginx指令的嵌套问题
1. ngx_http_init_locations
    ngx_queue_sort(locations, ngx_http_cmp_locations);
    ngx_queue_split(locations, q, &tail);
    ngx_queue_split(locations, named, &tail);
    ngx_queue_split(locations, regex, &tail);


## 三.nginx的location查找算法


