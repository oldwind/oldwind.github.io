---
layout: content
title: nginx源码五(4)：nginx配置文件加载-request阶段处理机制
status: 3
complete: 10% 
category: nginx
---

## 一.前言

在前面，分析了location指令配置信息的存储和查找流程，下面，重点分析的是nginx中对于request的分阶段处理的设计，但是，为什么要对一个request请求分阶段处理呢？

举个栗子，我们去医院看眼科疾病，我们可能经历什么样的一个流程呢？

1. 我们去挂号区挂号，拿到一个病例本
2. 到眼科看医生。看见到医生之前做了一系列工作
    - 到眼科负责的视力测试区，测了一下裸眼视力，被通知到下一个环节，做瞳孔散射测试
    - 到眼科负责的瞳孔散射区，做了瞳孔的散射，结束后，通知到下一个环节，见医生
3. 见到医生，医生看完后，觉得是病理性问题，让去化验科做一个化验
4. 去化验科化验
5. 拿着化验结果找眼科医生
6. 医生根据结果，开药
7. 去药房拿到药
8. 结束看病

这里面其实有几点需要我们思考的：
- 为什么需要一个病例本？
- 为什么分成挂号、眼科、化验科、药房等不同区域？
- 一个从没去过医院的患者，如何穿梭在不同的科室之间？
- 某个科室下面增加一个新的流程怎么办，例如，眼科医生看前，假设增加了一个色盲的检测，患者如何从容的走流程？

其实思考清楚这些问题，基本上，对于nginx处理request请求，有一个非常好的理解；
- `病例本` 是各个科室之间消息传递的载体，对于nginx来说，它构造了一个request对象，来处理reuqest请求，以及记录在各个阶段处理的结果
- `划分区域` 是划繁为简，分工明确，如果不分科室，一个患者来了，







## 二.流程
1. 创建
2. module分析后， postconfig
3. 初始化
4. 执行

{% highlight c %}
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
{% endhighlight %}

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




