---
layout: content
title: 谨慎使用php的strtotime
status: 1 
category: php
author:     "yimuren"
tags:
    - php
---


## 一. 业务场景

我们在日常业务中，针对业务量，经常会采用对数据库按时间做横向分表，分表后的查询往往会涉及到时间问题。例如，我们想查询某个用户距离当前时间1个月的订单情况，在这个时候，我们有些会用到strtotime()函数去处理。但是使用strtotime()，需要非常谨慎。我们先看一段代码，代码目的是想拿到几个月以前的年份月份，例如今天是2014年8月1号，我想拿到2个月前的年份月份是 array("0"=>"201406", "1"=>"201407",)

{% highlight php%} 
<?php
/*
*$mthNum 几月以前
* return like array('0'=>'201401','1'=>'201402')，结果不包含当前月份
*/
function getTimeYm($mthNum)
{
    $timeArr = array();
    if($mthNum <= 0)
        return $timeArr;
    
    do {
        $timeArr[] = date("Ym", strtotime("-$mthNum month"));
        $mthNum --;
    } while ($mthNum > 0);

    return $timeArr;
}
{% endhighlight %} 

表面看代码似乎没有问题，但是我们做个测试，下面是测试代码，测试的目的很简单，只是想测试一下，每个月最后一天的前一个月的日期是多少
{% highlight php%} 
<?php
$dateArr = array(
    "2014-01-31    00:00:00 -1 month",
    "2014-02-28    00:00:00 -1 month",
    "2014-03-31    00:00:00 -1 month",
    "2014-04-30    00:00:00 -1 month",
    "2014-05-31    00:00:00 -1 month",
    "2014-06-30    00:00:00 -1 month",
    "2014-07-31    00:00:00 -1 month",
    "2014-08-31    00:00:00 -1 month",
    "2014-09-30    00:00:00 -1 month",
    "2014-10-31    00:00:00 -1 month",
    "2014-11-30    00:00:00 -1 month",
    "2014-12-31    00:00:00 -1 month",
);

foreach ($dateArr as $val) {
    $time = strtotime($val);
    echo $val . " is:" . date("Y-m-d", $time). "\r\n";
}

{% endhighlight %} 

{% highlight bash%} 
2014-01-31  00:00:00 -1 month is:2013-12-31
2014-02-28  00:00:00 -1 month is:2014-01-28
2014-03-31  00:00:00 -1 month is:2014-03-03
2014-04-30  00:00:00 -1 month is:2014-03-30
2014-05-31  00:00:00 -1 month is:2014-05-01
2014-06-30  00:00:00 -1 month is:2014-05-30
2014-07-31  00:00:00 -1 month is:2014-07-01
2014-08-31  00:00:00 -1 month is:2014-07-31
2014-09-30  00:00:00 -1 month is:2014-08-30
2014-10-31  00:00:00 -1 month is:2014-10-01
2014-11-30  00:00:00 -1 month is:2014-10-30
2014-12-31  00:00:00 -1 month is:2014-12-01
{% endhighlight %} 

我们看一下测试结果，从测试结果中，我们发现我们忽略了每个月天数不同，那么strtotime()会带来不一样的结果，通过这个我们发现原来strtotime("-$n month")是这样计算的 (注：strtotime("-$n month")，第二个参数省略，第二个参数表示距离的时间，省略表示当前时间), 那么如果这样的话，我们怎么用strtotime("-$n month")处理我们的需求呢？ 下面提供一段手写代码供参考

{% highlight php%} 
/****************
*解决两个时间段之间的年月
* $btm, $etm 是unix时间戳
*****************/
function getTimeDis($btm, $etm)
{
    $resArr = array();
    if($etm < $btm)
        return $resArr;

    //将btm和etm都转成每月1号
    $btmc = strtotime(date("Y-m-01 00:00:00", $btm));
    $etmc = strtotime(date("Y-m-01 00:00:00", $etm));


    $flag = 0; //时间差标识符
    $resArr[] = date("Ym", $etmc);

    while(1) {
        $flag ++;
        $compTime = strtotime("-{$flag} month",  $etmc);
        if($compTime < $btm)
            break;

        $resArr[] = date("Ym", $compTime);
    }

    return array_unique($resArr);
}
{% endhighlight %} 

## 二. 深入研究
