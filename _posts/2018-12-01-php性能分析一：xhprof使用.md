---
layout: content
title: php性能分析一：xhprof使用说明
status: 1
# complete: 10% 
author:   "yimuren"
tags:
    - php
    - c
---

## 一. 背景 ##
xhprof是对php的性能进行分析的一个重要工具，当我们用strace满足不了需求的时候，对于php的性能分析，还是要用比较专业的分析统计工具

## 二. xhprof的安装 ##

xhprof是php的扩展
```bash#1. php7的xhprof的
git clone git@github.com:oldwind/php-xhprof-extension.git

#2. 进入下载目录
cd php-xhprof-extension

#3. 执行phpize
/php-7.0.29/bin/phpize

#4. 执行configue
./configure --with-php-config=/php-7.0.29/bin/php-config --prefix=/php-7.0.29/

#5. make
make && make install

#6. php.ini中指定扩展
[xhprof]
extension=tideways_xhprof.so

```

## 三. php.ini失效的问题 ##
在安装好扩展之后，突然发现在用php的终端命令执行的时候，报了fatal，显示function不存在，在这里php需要指定php.ini的文件地址。否则不会加载php.ini，原因做一个备忘，后面研究一下php内核中php.ini的加载机制
```bash
~/Debug/php-7.0.29/bin/php -c /Debug/php-7.0.29/etc/php.ini test_cd.php
```

## 四. 测试脚本 ##
简单的写一个测试脚本，我们看看xhprof的分析

```php<?php
echo microtime();
function test() {
    $m = 0;
    for ($i = 0; $i < 100; $i ++) {
        $m += $i;
    }
    return $m;
}

// 性能分析开始
tideways_xhprof_enable(TIDEWAYS_XHPROF_FLAGS_MEMORY | TIDEWAYS_XHPROF_FLAGS_CPU);

exec("ls -l");
test();

echo "\r\n";
echo microtime();
echo "\r\n";
// 返回性能分析的结果
$result = tideways_xhprof_disable();


$header = sprintf("%40s%10s%10s%10s%10s%10s", "name", "ct", "wt", "cpu", "mu", "pmu \r\n");
echo $header;
foreach($result as $key=>$val) {

    $body = sprintf("%40s%10s%10s%10s%10s%10s", $key, $val["ct"], $val["wt"], $val["cpu"], $val["mu"], "{$val["pmu"]}\r\n");
    echo $body;
}

```

我们来看一下结果

```bash
0.48859000 1547004996
0.49623300 1547004996
                                    name        ct        wt       cpu        mu    pmu
                           main()==>exec         1      7286       495       288       0
                      main()==>microtime         1        36        36       240       0
                                  main()         1      7524       642       928       0
                           main()==>test         1        36        36       160       0
        main()==>tideways_xhprof_disable         1        15        16       160       0
```

- `name`，调用的函数名
- `ct`， 函数执行的次数
- `wt`， 函数执行消耗的时间(微秒)
- `cpu`，函数消耗的cpu时间(微秒)
- `mu`， 父函数调用所有该子函数消耗的内存总和
- `pmu`，父函数调用所有该子函数消耗的内存峰值

我们发现脚本里面的请求大部分的消耗在`exec`上，一次shell调用，需要创建管道，需要起进程，性能损耗还是很明显的。 后面在仔细研究一下xhprof的实现原理。


