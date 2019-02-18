---
layout: content
title: php的base_convert方法溢出问题
status: 1
complete: 100% 
category: php
---

## 一. 背景

在进制转换中用到了php的base_convert方法，面对`大数`的时候踩到了一些坑，实际上用之前百度了一下这个方法，看到的是w3school(http://www.w3school.com.cn/php/func_math_base_convert.asp)里面的方法使用示例和说明

`base_convert(number,frombase,tobase)`

对于这个方法，使用说明是，"返回一个字符串，包含 number 以 tobase 进制的表示。number 本身的进制由 frombase 指定。frombase 和 tobase 都只能在 2 和 36 之间（包括 2 和 36）。高于十进制的数字用字母 a-z 表示，例如 a 表示 10，b 表示 11 以及 z 表示 35。"

`number`在使用过程中，可以是字符串，也可以是数字，由于高于十进制的数字用字母 a-z 表示，所以，一般情况下，`number`用做字符串来做参数写入，在进制转换中为什么有溢出情况呢，这个具体和`base_convert`这个方法的实现有关系，下面从源码角度，看一下这个方法的实现


## 二. 源码分析

### 2.1 主函数
从代码中，我们能看出几点，一是输入的number参数是一个zval格式，支持多种类型输入，输入的结果会被convert成string，同时，frombase和tobase支持的进制范围在2到36之间，也就是 `base_convert`支持2到36进制之间的数据的进制转换；类型处理完后的第二步，是将`number`转成10进制，转成的10进制数据，会根据是否大于long类型，放在zval类型的temp字段中；最后第三步，是将long类型或者double类型的number转换成tobase的进制。 下面是代码过程

{% highlight c %}
PHP_FUNCTION(base_convert)
{
	zval **number, temp;
	long frombase, tobase;
	char *result;

	if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "Zll", &number, &frombase, &tobase) == FAILURE) {
		return;
	}
	convert_to_string_ex(number);
	
	if (frombase < 2 || frombase > 36) {
		php_error_docref(NULL TSRMLS_CC, E_WARNING, "Invalid `from base' (%ld)", frombase);
		RETURN_FALSE;
	}
	if (tobase < 2 || tobase > 36) {
		php_error_docref(NULL TSRMLS_CC, E_WARNING, "Invalid `to base' (%ld)", tobase);
		RETURN_FALSE;
	}

	if(_php_math_basetozval(*number, frombase, &temp) == FAILURE) {
		RETURN_FALSE;
	}
	result = _php_math_zvaltobase(&temp, tobase TSRMLS_CC);
	RETVAL_STRING(result, 0);
} 
{% endhighlight %}


### 2.2 将number转成10进制
{% highlight c %}
PHPAPI int _php_math_basetozval(zval *arg, int base, zval *ret)
{
	....

	cutoff = LONG_MAX / base;
	cutlim = LONG_MAX % base;
	
	for (i = Z_STRLEN_P(arg); i > 0; i--) {
		c = *s++;

		/* might not work for EBCDIC */
		if (c >= '0' && c <= '9') 
			c -= '0';
		else if (c >= 'A' && c <= 'Z') 
			c -= 'A' - 10;
		else if (c >= 'a' && c <= 'z') 
			c -= 'a' - 10;
		else
			continue;

		if (c >= base)
			continue;
		
		switch (mode) {
		case 0: /* Integer */
			if (num < cutoff || (num == cutoff && c <= cutlim)) {
				num = num * base + c;
				break;
			} else {
				fnum = num;
				mode = 1;
			}
			/* fall-through */
		case 1: /* Float */
			fnum = fnum * base + c;
		}	
	}

	if (mode == 1) {
		ZVAL_DOUBLE(ret, fnum);
	} else {
		ZVAL_LONG(ret, num);
	}
	return SUCCESS;
}
{% endhighlight %}


### 2.3  将转成的10进制数放在zval的 lval，或者 dval变量中保存
{% highlight bash %}
(gdb) ptype arg->value
type = union _zvalue_value {
    long lval;
    double dval;
    struct {
        char *val;
        int len;
    } str;
    HashTable *ht;
    zend_object_value obj;
}
{% endhighlight %}

### 2.4 `_php_math_zvaltobase`转成对应的进制。 

这里不在详细叙述，这里我们可以看出number如果是一个大数，在进行进制转换的时候，明显会有溢出的问题。那么我们怎么解决这个问题呢


## 三. 处理方案

针对10进制转16进制，可以采用 http://php.net/manual/zh/book.bc.php 扩展库做大数运算，其它大数的进制转换，目前看没有好的办法，需要自己通过扩展实现，或者php语言本身去实现，php语言去实现效率要差一些。

后续有时间在写一个专用大数进制转换的扩展库。
