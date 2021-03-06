---
layout: content
title: nginx源码三(1)：nginx数据结构(ngx_array_t)
status: 1
complete: 10% 
author:     "yimuren"
tags:
    - nginx
---

## 一. 前言
数组是一种常见的数据结构，具备下面几个特点 [百度百科-数组]：
1. 相同数据类型的元素的集合。
2. 各元素的存储是有先后顺序的，它们`在内存中按照这个先后顺序连续存放在一起`。
3. 数组元素用整个数组的名字和它自己在数组中的顺序位置来表示。例如，a[0]表示名字为a的数组中的第一个元素，a[1]代表数组a的第二个元素，以此类推。

数组的存储方式有栈存储和堆存储，这里的`ngx_array_t`将的是在堆中的存储方式


## 二. 数据结构

nginx对于堆中实现数组和前面讲的内存池有共同的一个特点，就是对于申请到的一块连续内存，采用`头部+数据区`的的方式，我们先看下面的一张简图：

![nginx]({{site.baseurl}}/img/nginx/ngx_array1.jpg)

**需要说明一下，就是`头部`和 `数据区`不一定在连续内存上**，这张图是简画成了连续内存， 看到这张图，我们应该可以考虑到这个头部应该包含的数组元素，下面看一下详细的数据结构：

```c
typedef struct {
    void        *elts;
    ngx_uint_t   nelts;
    size_t       size;
    ngx_uint_t   nalloc;
    ngx_pool_t  *pool;
} ngx_array_t;
```

元素的基本含义是这样的：
- `*elts` 指向申请数组内存的起始位置
- `nelts` 记录创建的内存使用了多少
- `size` 数组的元素的大小
- `nalloc` 数组元素的个数
- `*pool` 指向内存池的起始位置

## 三. 基础算法

### 3.1 数组内存的释放 ngx_array_destroy
ngx_array_destroy 并不是要和操作系统交互释放内存，原因是，nginx释放array的内存是建立在数组分配在堆上，每次array的申请并不是单独申请内存，所以不能释放整个内存，nginx在处理的时候，通过last指针的方式来释放内存，内存释放考虑两点
1. 数据区的释放，如果数组的数据区的尾指针就是内存池可用内存的头指针，那么可以移动last指针来释放
2. 数据头的释放，条件同上

如果不符合上述条件，ngx_array_destory并不释放内存

### 3.2 数组增加元素 ngx_array_push(ngx_array_t *a)
- **input**: 需要push的数组指针
- **output**: 返回可以存储元素的地址

步骤：
- 1、检查`数组a`的数据区是否有空余位置 (nelts == nalloc)
    - 1.1、如果有空余位置，nelts 计数加1， 返回可存储的地址
    - 1.2、如果数据区无空余位置，查看能否在当前`内存条`上扩容，并且保持`连续`
        - 1.2.1、如果可以继续分配，则继续分配后返回
        - 1.2.2、不能继续分配的话，`从新申请一块2倍于原来数组元素大小的内存`，拷贝原来数组数据(原来的数组不释放)
- 2、返回结果



