---
layout: content
title: php扩展出现的一次跨平台问题
status: 1
category: go, php
author:   "yimuren"
tags:
    - go
---

## 一. 问题描述

开发的php扩展在新环境报错，究其原因是扩展中获取字符串长度出现了问题，看一下如下代码(代码做了名称相关的替换)

代码的思路比较简单，通过 `zend_parse_parameters` 方法，获取输入参数 `p0`, `p1`， 通过 `p0_len` 存储参数 `p0` 字符串的长度，这种方式在开发环境运行都是正常的，但是在docker里面运行的时候，出现了问题

- `p0 字符串有值，但是 p0_len 的长度为 0`

```c
//  __construct
PHP_METHOD(Test, __construct)
{
    const char *p0;
   	const char *p1;

	int p0_len, int p1_len;

	if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "ss", 
        &p0, &p0_len, 
        &p1, &p1_len) == FAILURE)
	{
		RETURN_FALSE;
	}

    ...
}
```

原因是什么？关键出现在 `int` 上，这是个不支持跨平台的参数类型

我们先一点一点梳理：

## 二. 关于 zend_parse_parameters

Zend内核提供了这个方法供php扩展获取上下文的参数输入：

```c
// num_args 参数个数
// type_spec 参数类型
ZEND_API int zend_parse_parameters(int num_args, const char *type_spec, ...)
```

一个函数的参数具备下面两个特点

1. 参数个数可变
2. 参数类型不定

解决参数可变运用的是c语言中解决变参问题的一组宏，所在头文件：`#include <stdarg.h>`，用于获取不确定个数的参数,

- `va_list`变量，这个变量是指向参数的指针
- `va_start`函数，初始化 `va_list` 变量
- `va_arg`函数返回可变的参数，va_arg 的第二个参数是要获取的参数的类型；va_arg在获取到参数后，会把 va_list变量指针向下移动一个位置
- `va_end` 函数结束可变参数的获取

```c
ZEND_API int zend_parse_parameters(int num_args, const char *type_spec, ...) 
{
	va_list va;
	int retval;
	int flags = 0;

	va_start(va, type_spec); // 从第二个参数开始，获取不定项的参数
	retval = zend_parse_va_args(num_args, type_spec, &va, flags);
	va_end(va);

	return retval;
}
```

在进程中,堆栈地址是从高到低分配的.当执行一个函数的时候,将参数列表入栈,压入堆栈的高地址部分,然后入栈函数的返回地址,接着入栈函数的执行代码,这个入栈过程,堆栈地址不断递减。
同时，堆栈中,各个参数的分布情况是倒序的.即最后一个参数在列表中地址最高部分,第一个参数在列表地址的最低部分

了解完获取不定参数的问题后，下面的问题是关于参数的类型，Zend内核支持不同的字符来表示参数的类型，下面是一个对应参数的类型表，挑选的是其中比较常用的部分

| 参数 | 代表着的类型 | 对应C里的数据类型|
|--|--|--|
| b | Boolean | zend_bool|
| l | Integer  | long   |
| d | Float    | double |
| s | String   | char * |
| r | Resource | zval * |
| a | Array | zval *|
| o | Object | zval * |
| O | 特定类型的Object | zval *, zend_class_entry* |
| z | 任意类型 | zval * |
| Z | zval**类型 | zval * |
| f | 表示函数、方法名称 | |
 
除了字符外，zend内核还使用一些符号来实现php的扩展函数的一些特性 

1. `*`  和 `+` 来实现扩展函数的参数不定问题, 该符号只能放在最后
2. `！` 用来对 `l`, `L`, `d`, `b` 类型获取是否为空的判断
3. `/`  主要针对 zval 类型是否是引用的问题
4. `|`  用来处理参数的不定类型可以..。


回到 `zend_parse_parameters` 这个函数，第一个参数表示扩展函数的入参的个数，第二个表示入参的类型，这个类型是 字符 + 符号来表示，例如

- sss, s+, s*， z/

.etc

我们在这里看看针对字符串类型的处理办法， 先看一下代码, 细节部分不再赘述，具体看看调用栈

```bash
(gdb) bt
#0  zend_parse_arg_impl (arg_num=1, arg=0x7ffff6815290, va=0x7fffffffaad0, spec=0x7fffffffaa30, error=0x7fffffffa9c0, severity=0x7fffffffa9bc) at /home/work/php-src-PHP-7.0.32/Zend/zend_API.c:552
#1  0x0000000000657661 in zend_parse_arg (arg_num=1, arg=0x7ffff6815290, va=0x7fffffffaad0, spec=0x7fffffffaa30, flags=0) at /home/work/php-src-PHP-7.0.32/Zend/zend_API.c:753
#2  0x0000000000657de2 in zend_parse_va_args (num_args=3, type_spec=0x7ffff65fe665 "ssss", va=0x7fffffffaad0, flags=0) at /home/work/php-src-PHP-7.0.32/Zend/zend_API.c:925
#3  0x0000000000657ff1 in zend_parse_parameters (num_args=4, type_spec=0x7ffff65fe665 "ssss") at /home/work/php-src-PHP-7.0.32/Zend/zend_API.c:959
#4  0x00007ffff65f8cb4 in zim_XuperchainV3___construct (execute_data=0x7ffff6815230, return_value=0x7ffff68151f0)
    at /home/work/icode/baidu/blockchain/lcv-sdk-php/xuperchainv3/.tmp/php7.0/xchainv3/src/php_xchainv3.cpp:292
#5  0x00000000006b233c in ZEND_DO_FCALL_SPEC_HANDLER (execute_data=0x7ffff68150f0) at /home/work/php-src-PHP-7.0.32/Zend/zend_vm_execute.h:842
#6  0x00000000006b1135 in execute_ex (ex=0x7ffff6815030) at /home/work/php-src-PHP-7.0.32/Zend/zend_vm_execute.h:417
#7  0x00000000006b125f in zend_execute (op_array=0x7ffff6881000, return_value=0x0) at /home/work/php-src-PHP-7.0.32/Zend/zend_vm_execute.h:458
#8  0x0000000000653aa9 in zend_execute_scripts (type=8, retval=0x0, file_count=3) at /home/work/php-src-PHP-7.0.32/Zend/zend.c:1445
#9  0x00000000005bff56 in php_execute_script (primary_file=0x7fffffffe2e0) at /home/work/php-src-PHP-7.0.32/main/main.c:2516
#10 0x0000000000722cca in do_cli (argc=2, argv=0xb818e0) at /home/work/php-src-PHP-7.0.32/sapi/cli/php_cli.c:977
#11 0x0000000000723c04 in main (argc=2, argv=0xb818e0) at /home/work/php-src-PHP-7.0.32/sapi/cli/php_cli.c:1347
```

如下，扩展针对字符串的处理，通过va_arg拿到需要设定字符串长度的传入参数指针，php内核是按照 `size_t` 的类型去设置长度值，而获取却是通过`int`去获取， `size_t` 在系统中是8位，而 `int` 在系统中是4位，导致设定 值的时候，出现了覆盖。

```c
    case 's':
    {
		char **p = va_arg(*va, char **);
		size_t *pl = va_arg(*va, size_t *);
		if (!zend_parse_arg_string(arg, p, pl, check_null)) {
			return "string";
		}
	}
	break;
```

```c
static zend_always_inline int zend_parse_arg_string(zval *arg, char **dest, size_t *dest_len, int check_null)
{
	zend_string *str;

	if (!zend_parse_arg_str(arg, &str, check_null)) {
		return 0;
	}
	if (check_null && UNEXPECTED(!str)) {
		*dest = NULL;
		*dest_len = 0;
	} else {
		*dest = ZSTR_VAL(str);
		*dest_len = ZSTR_LEN(str);
	}
	return 1;
}
```

## 三. 后记 ##

下面是一个例子，可以深入理解一下，获取值的时候，类型不同带来的问题。 

```c
#include <stdarg.h>
#include <stdio.h>
#include <stdlib.h>
// #include <_size_t.h>


int sum(int, ...);

int sum(int num_args, ...)
{
    int val = 0;
    va_list ap;
    int i;

    long *tmp;
    va_start(ap, num_args);
    for (i = 0; i < num_args; i++)
    {
        tmp = va_arg(ap, long *);
        // val += va_arg(ap, int *);
        printf("#####---%ld\n", *tmp);
    }

    va_start(ap, num_args);
    int *temp;
    for (i = 0; i < num_args; i++)
    {
        temp = va_arg(ap, int *);
        // val += va_arg(ap, int *);
        printf("#####---%d\n", *temp);
    }

    va_end(ap);

    return val;
}

int main()
{
    int a, b;
    int *p, *q;
    a = 5;
    b = 6;
    p = &a;
    q = &b;
    printf("15 和 56 的和 = %d\n", sum(2, p, q));
    return 0;
}
```