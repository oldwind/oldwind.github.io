---
layout: content
title: 一个php变量名问题的追查之路
status: 3
complete: 10% 
category: php
---

## 一.背景

从php的基础文档里面，我找到了php的变量命名规则，

- `变量名与 PHP 中其它的标签一样遵循相同的规则。一个有效的变量名由字母或者下划线开头，后面跟上任意数量的字母，数字，或者下划线。按照正常的正则表达式，它将被表述为：'[a-zA-Z_\x7f-\xff][a-zA-Z0-9_\x7f-\xff]*'。`

之前从来没有考虑过php的变量命名问题，因为习惯了`字母+数字+"_"`, 基本就是这几种组合着来，而这次，遇到了个奇怪的bug，我们来看一下代码;

![php-varchar-20190129]({{site.baseurl}}/img/php/php-varchar-20190129.png)

简单的看，两行代码是一模一样的，都是对一个变量赋了值，然后`var_dump`出来， 那么为什么一个返回值是 `NULL`， 另外一个是正常呢，具体的原因是什么？实际上在Mac下面，这个错误很容易出现，在Mac下敲空格的时候，如果不小心同时摁了 `option` 键，就会产生一个 UTF-8的空格，我们代码的编码方式一般是采用UTF-8编码，根据前面的php变量命名规则，这个空格会被算到php的变量中，所以导致的第一个的NULL， 下面我们查看一下脚本编码

## 二. 分析之路1-脚本编码分析

{% highlight bash %}
00000000: 3c3f 7068 700a 2468 65c2 a03d 2022 6865  <?php.$he..= "he
00000010: 6c6c 6f22 3b20 0a76 6172 5f64 756d 7028  llo"; .var_dump(
00000020: 2468 6529 3b0a 0a24 6865 203d 2022 6865  $he);..$he = "he
00000030: 6c6c 6f22 3b0a 7661 725f 6475 6d70 2824  llo";.var_dump($
00000040: 6865 293b 0a0a 0a                        he);...
{% endhighlight %}

我们仔细看一下编码
- 第一个`he`变量后是两个字节，对应的分别是 `c2 a0`，这个在UTF-8上识别成空格， 而C2 =  (12 * 16 +2) = 198; a0 = 160这个会通过php的变量检查
- 第二个`he`变量后面是一个字节，其实就是asicc码下面的空格符号

看到这里，其实就很明白产生的原因了，utf8的编辑器识别的utf8空格和asicc空格是一样的，php在解析的时候，把这个空格算到php变量的一部分了。

但是，我并不想在这里停止了，我更感兴趣的是php内核对变量的处理，回忆之前对php内核的理解，处理php代码有几个大的步骤，
1. 配置文件的解析
2. extension的加载
3. 文件的编译，采用了linux的Lex进行了词法分析，采用linux的Bison进行语法分析 生成opcode
4. opcode在Zend虚拟机内执行
5. 返回结果

我们这次追查的核心是变量规则的分析阶段，具体的代码在哪里呢，我们用gdb来走走调试之路


## 三. 分析之路2-对php内核的分析

接着前面的php内核处理脚本流程，我们能感觉，无非两个阶段，"编译"期间或者"执行"期间； 这里叉开一下话题，我尝试用Go写了一个测试，赋值给一个非法变量名的变量，Go的编译期是过不去，php的编译和执行是在一起的，但是我们可以猜想变量的分析，是否合法是在编译阶段，用gdb调试可以明确这一点

{% highlight bash %}
Thread 2 hit Breakpoint 2, zend_execute_scripts (type=8, retval=0x0, file_count=3) at Zend/zend.c:1321
1321			EG(active_op_array) = zend_compile_file(file_handle, type TSRMLS_CC);
(gdb) bt
#0  zend_execute_scripts (type=8, retval=0x0, file_count=3) at Zend/zend.c:1321
#1  0x00000001005c358b in php_execute_script (primary_file=0x7ffeefbff140) at main/main.c:2502
#2  0x000000010071fa17 in do_cli (argc=2, argv=0x7ffeefbff858) at sapi/cli/php_cli.c:989
#3  0x000000010071e8de in main (argc=2, argv=0x7ffeefbff858) at sapi/cli/php_cli.c:1365
(gdb) n

Parse error: syntax error, unexpected '=' in /test.php on line 2
[Inferior 1 (process 74169) exited with code 0377]
{% endhighlight %}
  
{% highlight bash %}
#0  zendparse () at Zend/zend_language_parser.c:6869
#1  0x00000001006080d2 in compile_file (file_handle=0x7ffeefbff140, type=8) at Zend/zend_language_scanner.l:585
#2  0x0000000100343c21 in phar_compile_file (file_handle=0x7ffeefbff140, type=8) at ext/phar/phar.c:3411
#3  0x000000010066780f in zend_execute_scripts (type=8, retval=0x0, file_count=3) at Zend/zend.c:1321
#4  0x00000001005c358b in php_execute_script (primary_file=0x7ffeefbff140) at main/main.c:2502
#5  0x000000010071fa17 in do_cli (argc=2, argv=0x7ffeefbff858) at sapi/cli/php_cli.c:989
#6  0x000000010071e8de in main (argc=2, argv=0x7ffeefbff858) at sapi/cli/php_cli.c:1365
{% endhighlight %}

这个时候，我有点怀念php的调用栈了
