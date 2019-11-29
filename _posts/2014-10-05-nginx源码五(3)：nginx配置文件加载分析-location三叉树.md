---
layout: content
title: nginx配置文件分析-location前缀树
status: 1
complete: 10% 
author:     "yimuren"
recommend: 1
tags:
    - nginx
---

## 一.前言

我们在前面提到了nginx的指令分析，针对server {} 里面的 location {} 指令，平行的 location指令 和 嵌套的 location指令 之间都会通过 queue 的双向链表来进行串联， 双向链表的头节点分别是 location {} 的父 location指向的 ngx_http_core_moudle的配置文件区，如下图

![ngx_loc_tree_conf]({{site.baseurl}}/img/nginx/ngx_loc_data.jpeg)

上面的图只是location {} 指令的分析的初始化阶段，下一步，会经历多个步骤，最终构建一个location的静态树，方便nginx在处理http请求的时候，在找到server后，快速的找到location的配置，总的来说，分成下面几个步骤：
- 1.location配置的创建
- 2.location指令的排序
- 3.location指令的切分
- 4.location精确匹配和包含匹配的合并
- 5.处理location的包含关系构成list
- 6.location的三叉树(前缀树)形成

另外我们也去分析一下对于request请求的uri匹配一下

为了便于理解，我在这里设计了一个location的示例，我们看一下三叉树的形成过程
 
```c
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
```


## 二.location配置的创建

server{} 指令内部的location {} 指令，是一对多的关系，同时location {}内部还支持嵌套，nginx在分析完每一条指令后，会将配置信息写到创建的配置管理中，我们在开始的图中，描述了这个配置的关系，在配置信息创建后，下面就可以处理这些配置，走到下面第三步，对指令配置做一个排序。 为了方便理解，我们按照上面的 location的配置示例，画了下面一张图， 在这里，简单起见，我们不考虑 location内部的嵌套关系

![ngx_loc_tree_conf]({{site.baseurl}}/img/nginx/ngx_loc_init.jpeg)


## 三.location指令的排序

nginx通过queue去串联配置的location，下一步是对location进行排序，排序算法的实现，在前面关于ngx_queue_t的数据结构分析中，做了分析，简单说就是：
1. ngx_queue_t实现自己的排序算法，针对链表，nginx选择了`插入排序`的算法
2. 使用的业务实现自己的比较算法，nginx设计location指令先后关系的比较算法是 ngx_http_cmp_locations

我们看一下针对location指令的比较逻辑

```c
static ngx_int_t
ngx_http_cmp_locations(const ngx_queue_t *one, const ngx_queue_t *two)
{
    ...
    if (first->noname && !second->noname) {
        /* shift no named locations to the end */
        return 1;
    }

    ...
    rc = ngx_filename_cmp(first->name.data, second->name.data,
                          ngx_min(first->name.len, second->name.len) + 1);

    if (rc == 0 && !first->exact_match && second->exact_match) {
        /* an exact match must be before the same inclusive one */
        return 1;
    }

    return rc;
}
```

在这里，我们简单说一下这个比较算法实现的目标：
1. noname类型的location放在最后
2. named放在noname类型以前，同时named location之间有一个排序，排序原则按照 `asicc码顺序排序`
3. 如果支持正则，正则排在named类型以前
4. 包含类型和精确类型的location混排，但是保持有序，排序原则同时按照 `asicc码顺序排序`

`asicc码顺序排序`是我这边单起的名字，没有找到专业术语，这里解释一下，例如对于 a, ab 排序，a > ab 排序原则是，先比较首字母，都是a，继续比较第二个字母，不存在设定asicc的值是0，然后 0 - 'b' < 0 所以 a 在 ab 前面

下面一张图是针对前面的示例做的排序：

![ngx_loc_tree_conf]({{site.baseurl}}/img/nginx/ngx_loc_split1.jpeg)


## 四.location指令的切分

排完序以后，location指令实际被做了几次切分，分别是 (精确匹配 + 包含类型匹配) + 正则匹配 + named类型匹配 + nonamed类型匹配，nginx下一步是切分，切分后，通过上下文配置的不同变量来管理。 这里不做详细描述，切分后可以参考一下下面的图：

![ngx_loc_tree_conf]({{site.baseurl}}/img/nginx/ngx_loc_split2.jpeg)


## 五.location指令的精确匹配和包含匹配的合并

正则匹配和named匹配指令已经被切走了，那么剩余下来是 精确 + 包含类型匹配， 我们可以看一下下面的数据结构，做了注释

```c
typedef struct {
    ngx_queue_t                      queue;       // 平级location之间的双向链表
    ngx_http_core_loc_conf_t        *exact;       // 精确匹配 指向的 ngx_http_core_loc_conf_t 配置
    ngx_http_core_loc_conf_t        *inclusive;   // 包含关系指向的 ngx_http_core_loc_conf_t 配置
    ngx_str_t                       *name;        // location名字
    u_char                          *file_name;   // 配置文件名 
    ngx_uint_t                       line;  
    ngx_queue_t                      list;        // 包含关系的开链指针
} ngx_http_location_queue_t;
```

我们可以看到，`ngx_http_location_queue_t` 中可以支持精确和包含关系的 ngx_http_core_loc_conf_t 配置，所以，在这步，nginx做了一件是，就是合并，例如，示例中的 `location /` 和 `location =/` 合并到一个`ngx_http_location_queue_t` 中。

![ngx_loc_tree_conf]({{site.baseurl}}/img/nginx/ngx_loc_exac.jpeg)

## 六.location指令生成包含关系的list

做完上面的操作后，nginx针对包含关系，选择了开链的处理方式，如下图：

![ngx_loc_tree_conf]({{site.baseurl}}/img/nginx/ngx_loc_list.jpeg)

## 七.location的三叉树形成

终于到了最后一步，构造nginx 的location 三叉树。我们先看一下树节点的数据结构

```c
struct ngx_http_location_tree_node_s {
    ngx_http_location_tree_node_t   *left;  // assicc 值小
    ngx_http_location_tree_node_t   *right; // assicc 值大
    ngx_http_location_tree_node_t   *tree;  // 包含关系

    ngx_http_core_loc_conf_t        *exact;  
    ngx_http_core_loc_conf_t        *inclusive;

    u_char                           auto_redirect;
    u_char                           len;
    u_char                           name[1];
};
```

对于前面生成的list，nginx创建树，按照下面的流程递归处理：
1. 获取queue中的中间位置节点(q)作为根节点。切成三部分，第一部分是q的左边作为左子树，第二部分是q，第三部分是q的右边称为右子树，q的包含关系作为tree
2. 递归处理左子树、右子树、tree部分
3. 所有处理中，如果只剩下queue的节点，上图list的红色部分，return出来

下图则是示例的形成结果，实际上还有一点，就是，为了节约空间，树在存储的时候，按照前缀方式做了存储设计

![ngx_loc_tree_conf]({{site.baseurl}}/img/nginx/ngx_loc_tree.jpeg)


## 八.查找算法 ngx_http_core_find_static_location

前面主要分析了nginx处理location指令的流程，location的配置信息存储分成多个部分；实际上在接收到request请求的时候，nginx要找到对应处理的location，重点分成下面两部分
- 精确匹配规则和包含匹配规则在建立的location三叉树中。
- 正则匹配规则。

那么一个请求过来的时候，匹配的算法是怎么样的呢？重点可以看下面两处代码： 大致是这样的思路
- 从三叉树中查找，
    - 如果什么都查不到，返回 NGX_DECLINED
    - 如果查找到精确匹配的规则，配置信息赋值给request对象，返回 NGX_OK
    - 如果查到包含关系的匹配，配置信息赋值给request对象，返回 NGX_AGAIN，
        - 递归 处理 location 的嵌套查询
- 判断三叉树&嵌套查询结果，
    - 返回是 NGX_OK 或者 NGX_DONE 直接返回
    - 其它情况，继续流程
- 判断查找到的配置是否支持继续进行正则匹配
    - 不支持，返回前面结果
    - 支持，进行正则匹配，
        - 匹配成功
            - 递归处理正则匹配中嵌套的匹配结果
        - 未匹配到，返回错误
    
```c
/*
 * NGX_OK       - exact or regex match
 * NGX_DONE     - auto redirect
 * NGX_AGAIN    - inclusive match
 * NGX_ERROR    - regex error
 * NGX_DECLINED - no match
 */

static ngx_int_t
ngx_http_core_find_location(ngx_http_request_t *r)
{
    ngx_int_t                  rc;
    ngx_http_core_loc_conf_t  *pclcf;
#if (NGX_PCRE)
    ngx_int_t                  n;
    ngx_uint_t                 noregex;
    ngx_http_core_loc_conf_t  *clcf, **clcfp;

    noregex = 0;
#endif

    pclcf = ngx_http_get_module_loc_conf(r, ngx_http_core_module);

    rc = ngx_http_core_find_static_location(r, pclcf->static_locations);

    if (rc == NGX_AGAIN) {

#if (NGX_PCRE)
        clcf = ngx_http_get_module_loc_conf(r, ngx_http_core_module);

        noregex = clcf->noregex; // 设定是否继续进行正则匹配
#endif

        /* look up nested locations */

        rc = ngx_http_core_find_location(r);
    }

    if (rc == NGX_OK || rc == NGX_DONE) {
        return rc;
    }

    /* rc == NGX_DECLINED or rc == NGX_AGAIN in nested location */

#if (NGX_PCRE)

    if (noregex == 0 && pclcf->regex_locations) {

        for (clcfp = pclcf->regex_locations; *clcfp; clcfp++) {

            ngx_log_debug1(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                           "test location: ~ \"%V\"", &(*clcfp)->name);

            n = ngx_http_regex_exec(r, (*clcfp)->regex, &r->uri);

            if (n == NGX_OK) {
                r->loc_conf = (*clcfp)->loc_conf;

                /* look up nested locations */

                rc = ngx_http_core_find_location(r);

                return (rc == NGX_ERROR) ? rc : NGX_OK;
            }

            if (n == NGX_DECLINED) {
                continue;
            }

            return NGX_ERROR;
        }
    }
#endif

    return rc;
}
```


```c
static ngx_int_t
ngx_http_core_find_static_location(ngx_http_request_t *r,
    ngx_http_location_tree_node_t *node)
{
    u_char     *uri;
    size_t      len, n;
    ngx_int_t   rc, rv;

    len = r->uri.len;
    uri = r->uri.data;

    rv = NGX_DECLINED;

    for ( ;; ) {

        if (node == NULL) {
            return rv;
        }

        ngx_log_debug2(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                       "test location: \"%*s\"", node->len, node->name);

        n = (len <= (size_t) node->len) ? len : node->len;

        rc = ngx_filename_cmp(uri, node->name, n);

        if (rc != 0) {
            node = (rc < 0) ? node->left : node->right;

            continue;
        }

        if (len > (size_t) node->len) {

            if (node->inclusive) {

                r->loc_conf = node->inclusive->loc_conf;
                rv = NGX_AGAIN;

                node = node->tree;
                uri += n;
                len -= n;

                continue;
            }

            /* exact only */

            node = node->right;

            continue;
        }

        if (len == (size_t) node->len) {

            if (node->exact) {
                r->loc_conf = node->exact->loc_conf;
                return NGX_OK;

            } else {
                r->loc_conf = node->inclusive->loc_conf;
                return NGX_AGAIN;
            }
        }

        /* len < node->len */

        if (len + 1 == (size_t) node->len && node->auto_redirect) {

            r->loc_conf = (node->exact) ? node->exact->loc_conf:
                                          node->inclusive->loc_conf;
            rv = NGX_DONE;
        }

        node = node->left;
    }
}
```


单从代码看，理解还是比较困难，举个下面的例子，我们看看下面几个问题
1. `/cab2ms 会匹配到哪个location`
    - 返回 4

2. `如果/ca配置去掉， /cab2ms 会匹配到哪个location` 
    - 返回 9

```c

http {
    server {
        ## /c配置
        location ^~ /c {
            return 200 "9";
        }

        ## /ca配置
        location /ca {
            location ~ /cab[0-1]m* {

                location ~ /cab[0-7]ms* {
                    return 200 "1";
                }
                return 200 "2";
            }

            return 200 "3";
        }

        ## /cab[0-9]m* 配置
        location ~ /cab[0-9]m* {
            location ~ /cab[0-2]m* {
                return 200 "4";
            }

            return 200 "5";
        }
    }
} 
```


总体匹配还是 精确 > 正则 > 包含原则



## 九.总结

总的来说，nginx的location配置非常复杂，而且很难用语言概括，里面有些设计存在需要讨论的地方，例如，禁止正则匹配，`是否要具备继承性`，如果我们把包含关系理解成父子关系，那么包含关系的父节点禁止用正则匹配规则，子节点是否自动继承禁止用匹配呢

另外，一个请求发起到location {}，中间还要经历一层rewrite过程，rewrite是通过nginx的模块化来实现，在ngx_http_rewrite_module.c 中做了实现，我们在这里后面 <a href="/index.html#/nginx/2014/10/14/nginx源码十四(1)-nginx的rewrite模块分析.html" title="nginx源码十四(1)：nginx的rewrite模块分析">nginx源码十四(1)：nginx的rewrite模块分析</a> 在接着继续分析