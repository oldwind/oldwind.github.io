---
layout: content
title: nginx源码五(4)：nginx配置文件加载-request阶段处理机制
status: 3
complete: 10% 
author:     "yimuren"
tags:
    - nginx
---

## 一.前言

在前面，分析了location指令配置信息的存储和查找流程，下面，重点分析的是nginx中对于request的分阶段处理的设计，但是，为什么要对一个request请求分阶段处理呢？

举个栗子，我们去医院看眼科疾病，我们可能经历什么样的一个流程呢？

1. 我们去挂号区挂号，拿到一个病例本
2. 到眼科看医生。看见到医生之前做了一系列工作
    - 到眼科负责的视力测试区，测了一下裸眼视力，被通知到下一个环节，做瞳孔散射测试
    - 到眼科负责的瞳孔散射区，做了瞳孔的散射，结束后，通知到下一个环节，见医生
3. 见到医生，医生看完后，觉得是病理性问题，让去化验科做一个化验
4. 去交钱，然后拿着收条去化验科
5. 去化验科化验
6. 拿着化验结果找眼科医生
7. 医生根据结果，开药
8. 去药房拿到药
9. 结束看病

这里面其实有几点需要我们思考的：
- 为什么需要一个病例本？
- 为什么分成挂号、眼科、化验科、药房等不同科室？
- 一个从没去过医院的患者，如何穿梭在不同的科室之间？
- 某个科室下面增加一个新的流程怎么办，例如，眼科医生看前，假设增加了一个色盲的检测，患者如何从容的走流程？

其实思考清楚这些问题，基本上，对于nginx处理request请求，有一个非常好的理解；
- `病例本` 是各个科室之间消息传递的载体，对于nginx来说，它构造了一个request对象，来处理reuqest请求，以及记录在各个阶段处理的结果
- `划分科室` 一个系统，模块的功能性划分和医院的科室划分有着同样的道理，系统越大，模块化后，可以让更多的管理者参与到不同的模块
- `科室转换` 看病有流程，但是找到科室后，可能还会到别的科室，例如这里医生让找化验科的科室化验，这个流程在nginx上也有体现，这个模块处理完了，可能需要找到下一个模块，下一个模块谁来控制查找，当前处理模块最有决策权 


## 二.流程

理解完上面举的例子，对nginx的阶段性处理应该有个比较好的思路，nginx在处理request中，大体分成三个阶段
- 第一阶段是处理连接请求，解析header头部信息，查找到处理请求的server
- 第二阶段是启动阶段处理机制，整体上nginx分成了11个阶段，在这11个阶段中，有部分阶段开发者可以挂载自己开发的插件
- 第三阶段是返回处理结果的阶段。

我在下面列了一个图，表述在第二阶段的所做的阶段处理机制，这11个阶段大致要做的事情，其中有四个阶段是不能挂载扩展的，图中已经标示出来

![ngx_http_phase1]({{site.baseurl}}/img/nginx/ngx_http_phase1.png)





```c
void
ngx_http_core_run_phases(ngx_http_request_t *r)
{
    ngx_int_t                   rc;
    ngx_http_phase_handler_t   *ph;
    ngx_http_core_main_conf_t  *cmcf;

    cmcf = ngx_http_get_module_main_conf(r, ngx_http_core_module);

    ph = cmcf->phase_engine.handlers;

    while (ph[r->phase_handler].checker) {

        rc = ph[r->phase_handler].checker(r, &ph[r->phase_handler]);

        if (rc == NGX_OK) {
            return;
        }
    }
}
```

## 二.nginx指令的嵌套问题

nginx的指令嵌套重点关注两部分代码，一是

```c
struct ngx_command_s {
    ngx_str_t             name;
    ngx_uint_t            type;
    char               *(*set)(ngx_conf_t *cf, ngx_command_t *cmd, void *conf);
    ngx_uint_t            conf;
    ngx_uint_t            offset;
    void                 *post;
};
```




序号	阶段	功能	注册的模块
0	NGX_HTTP_POST_READ_PHASE		ngx_http_realip_module
1	NGX_HTTP_SERVER_REWRITE_PHASE	server级别的url重写阶段	ngx_http_rewrite_module
2	NGX_HTTP_FIND_CONFIG_PHASE	查找location阶段	
3	NGX_HTTP_REWRITE_PHASE	location级别的url重写阶段，可被执行多次	ngx_http_rewrite_module
4	NGX_HTTP_POST_REWRITE_PHASE		
5	NGX_HTTP_PREACCESS_PHASE		"ngx_http_degradation_module
ngx_http_limit_conn_module
ngx_http_limit_req_module
ngx_http_realip_module"
6	NGX_HTTP_ACCESS_PHASE		"ngx_http_access_module
ngx_http_auth_basic_module
ngx_http_auth_request_module
"
7	NGX_HTTP_POST_ACCESS_PHASE		
8	NGX_HTTP_TRY_FILES_PHASE		
9	NGX_HTTP_CONTENT_PHASE	内容生成阶段	
10	NGX_HTTP_LOG_PHASE	记录访问日志阶段	ngx_http_log_module

