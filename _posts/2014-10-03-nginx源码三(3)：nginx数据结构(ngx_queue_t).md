---
layout: content
title: nginx源码三(3)：nginx数据结构(ngx_queue_t)
status: 1
complete: 10% 
author:     "yimuren"
tags:
    - nginx
---

## 一.前言

队列是一种比较常用的数据结构，主要用于实现先进先出，我们通常情况下会用链表、环型数组或者两个栈来实现队列，

- 环形数组来实现队列，需要预先确定容量，在使用过程中不是很方便
- 双栈的话，多存在面试题中
- 使用链表比较长见，我们看一下下面一张图

![ngx_queue]({{site.baseurl}}/img/nginx/ngx_queue1.jpg)

可以用两个数据结构来实现先进先出，一个queue的头节点指向一个单向链表的头尾指针，另外一个单向链表。 通过头部增加节点，尾部删除节点的方法实现先进先出。 这种方法里面存在两个问题

1. 链表里面每个节点通过指针指向数据区，在遍历获取链表数据时候，会发生两次寻址才能找到数据，一次找节点，一次找数据。
2. 通过两个数据结构来标识队列，稍微复杂一些，不够优美

这里，在性能方面，nginx首先把节点和使用到queue的数据结构做了融合，通过计算结构体偏移量的方式，去获取队列里面的数据元素，其次nginx的queue采用了环形双向链表的结构，对于queue的操作会非常简单灵活。

![ngx_queue]({{site.baseurl}}/img/nginx/ngx_queue2.jpg)

<br/>

## 二. nginx定义和实现的queue

我们先看一下queue的数据结构定义，非常简明，就是两个指针，一个指向下一个节点的指针，一个指向上一个节点的指针，同时构成一个环形结构

```c
typedef struct ngx_queue_s  ngx_queue_t;

struct ngx_queue_s {
    ngx_queue_t  *prev;
    ngx_queue_t  *next;
};
```

下面一张图是队列的示例图，展示单个节点到多节点队列的形成过程，环形列表构成的队列，作用是很明晰的，外部只需要定义一个 `ngx_queue_t * `，就可以操作这个队列

![ngx_queue]({{site.baseurl}}/img/nginx/ngx_queue3.jpg)


## 三. ngx_queue_t的使用

我们选择一个使用了队列的结构体来说明一下，nginx中队列的使用，在nginx的配置分析中，关于location的配置信息使用到了队列， 我们可以看一下location的数据结构

```c
typedef struct {
    ngx_queue_t                      queue;
    ngx_http_core_loc_conf_t        *exact;
    ngx_http_core_loc_conf_t        *inclusive;
    ngx_str_t                       *name;
    u_char                          *file_name;
    ngx_uint_t                       line;
    ngx_queue_t                      list;
} ngx_http_location_queue_t;
```

![ngx_queue]({{site.baseurl}}/img/nginx/ngx_queue4.jpg)

`ngx_queue_t list;    ngx_queue_t queue;`， 作为`ngx_http_location_queue_t`的成员变量，首先说明如何获取其它成员信息

```c
#define ngx_queue_data(q, type, link)  (type *) ((u_char *) q - offsetof(type, link))
```

我们假设有一个成员变量是 ngx_http_location_queue_t 类型， 知道list的指针 `ngx_queue_t *`，如何获取 `name`信息呢，可以按照下面的代码
```c
// q 是 ngx_queue_t * 类型指针，在list位置存储
ngx_http_location_queue_t *data = ngx_queue_data(q, ngx_http_location_queue_t, list )
```


## 四. 总结

nginx的queue设计带来非常灵活的操作，操作方法都非常简单明晰，包括，queue的初始化、插入、删除、查找中间位置等等，这里不在详细叙述。总得来说，queue的设计还是带来新的思考，一种耳目一新的感觉
