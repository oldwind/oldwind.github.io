---
layout: content
title: nginx源码二：nginx内存分配
status: 1
author:     "yimuren"
tags:
    - nginx
---

## 一. 前言

Linux下，操作系统提供应用程序了 `malloc` `free`来分配内存，对于应用程序来说，如果采用需要了，就调用操作系统Api去分配内存，那么必然存在两个问题

1. 产生大量的内存碎片； 
2. 性能问题； 不断的上下文交互，势必影响应用的性能

解决这样的问题，有通用的方法，一般采用内存池的方式； 应用程序一般先申请较大的一块内存，大小可以是系统分页大小(避免跨页查询)， 需要内存的时候，从内存池中找；申请的内存不够用，可以继续申请一块大的，同时将这些内存串联起来统一管理；

下面看一下nginx的内存池设计

## 二. nginx的内存池设计

### 2.1 nginx内存池数据结构

```c
struct ngx_pool_s {
    ngx_pool_data_t       d;
    size_t                max;
    ngx_pool_t           *current;
    ngx_chain_t          *chain;
    ngx_pool_large_t     *large;
    ngx_pool_cleanup_t   *cleanup;
    ngx_log_t            *log;
};

typedef struct {
    u_char               *last;
    u_char               *end;
    ngx_pool_t           *next;
    ngx_uint_t            failed;
} ngx_pool_data_t;

```

我们用一张图来表示一下这张结构

![nginx]({{site.baseurl}}/img/nginx/ngx_pool1.jpg)

`内存池`实际上并没有`池`，所谓的池是由多个串联的`内存块`构成，如上图，我们把这个称作`内存块`，应用程序每次向操作系统申请这么大的一块`内存块`, 这个内存块又分成两部分，`内存块的头部区`和`内存块的数据区`， 头部区管理内存的`分配`和不同`内存块`之间的串联

其中 ` ngx_pool_data_t  d`，指向内存的数据区，各个字段表达的含义如下：

- *last 指向未使用内存的起始位置
- *end 指向支持内存使用的末尾
- *next下一块申请的内存起始地址
- failed，内存分配失败计数器， 失败5次，不在该内存块继续分配
- max，可以使用内存大小，可用内存不超过内存分页的大小
- *current 指向当前可以分配内存的`内存块`地址，内存块是一个链式结构，未避免遍历整个链，增加了这个字段
- *large 如果申请的内存超过统一的`内存块`大小，则单独申请 

### 2.2 nginx内存分配

内存分配涉及流程，通常情况下包含三个部分
- 第一部分是**初始化**: nginx在这里初始化一个`内存块`，后面不断生成新的`内存块`，通过链表方式，构建一个内存池
- 第二部分是**内存分配**: 这里的内存分配不是向操作系统申请内存分配，而是通过内存池的方式分配内存，内存池在分配内存的时候，关注内存的对齐，综合考虑内存分配的效率等问题
- 第三部分是**内存的回收**: 内存池是由多个内存块构成的链式结构，那么回收释放内存变的比较简单，可以遍历整个链表，逐一释放

内存池在分配内存时，做的内存对齐的优化，我们可以看一下
```c
void *
ngx_palloc(ngx_pool_t *pool, size_t size)
{
    u_char      *m;
    ngx_pool_t  *p;

    if (size <= pool->max) {
        p = pool->current;
        do {
            m = ngx_align_ptr(p->d.last, NGX_ALIGNMENT);
            if ((size_t) (p->d.end - m) >= size) {
                p->d.last = m + size;
                return m;
            }
            p = p->d.next;
        } while (p);
        return ngx_palloc_block(pool, size);
    }
    return ngx_palloc_large(pool, size);
}

```

其中实现内存对齐的函数 `ngx_align_ptr`定义如下：

```c #define ngx_align_ptr(p, a)                                                   \
    (u_char *) (((uintptr_t) (p) + ((uintptr_t) a - 1)) & ~((uintptr_t) a - 1))
```

`对于内存块，有空余内存，但是，如果每次申请内存都比空余内存块大，怎么办？`

这里面有两个取舍
1. 内存块的充分利用
2. 内存分配效率

`充分利用`，那么每次申请都应该尝试查看一下内存块的内存是否够用； `分配效率高`，如果发现内存一次不够，直接将current指针指向下一个内存块； 这两个是矛盾的，nginx做了一个取舍，如果发现内存块分配失败的次数超过了`4`次，则不在分配

```c
static void *
ngx_palloc_block(ngx_pool_t *pool, size_t size)
{
    u_char      *m;
    size_t       psize;
    ngx_pool_t  *p, *new;

    psize = (size_t) (pool->d.end - (u_char *) pool);

    m = ngx_memalign(NGX_POOL_ALIGNMENT, psize, pool->log);
    if (m == NULL) {
        return NULL;
    }

    new = (ngx_pool_t *) m;

    new->d.end = m + psize;
    new->d.next = NULL;
    new->d.failed = 0;

    m += sizeof(ngx_pool_data_t);
    m = ngx_align_ptr(m, NGX_ALIGNMENT);
    new->d.last = m + size;

    for (p = pool->current; p->d.next; p = p->d.next) {
        if (p->d.failed++ > 4) {
            pool->current = p->d.next;
        }
    }

    p->d.next = new;

    return m;
}
```

### 2.3 内存池的总体预览
综合上面，通过下图可以看到一个内存池的总体结构

![nginx]({{site.baseurl}}/img/nginx/ngx_pool2.jpg)


## 三. 总结

nginx的内存池是一个比较简单的结构，在操作系统和应用程序之间做了很薄的一层，但作用却是非常大，一方面，减少了内存碎片； 同时，减少了内核和应用程序的上下文切换，提升内存分配的性能； 还有一点就是，封装系统的内存分配方法后，可以更简单的实现跨平台。


