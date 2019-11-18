---
layout: content
title: Function registration failed - duplicate name
status: 2 
category: go, php
author:   "yimuren"
tags:
    - go
---

## 一. 前言

编写的php扩展在运行时报了 "Warning: Function registration failed - duplicate name"的错误，错误报完后，直接core dump了，我们知道，php的函数是以hash表的形式存储的，很明显出现了相同的key值，在注册函数的时候出现了错误，实际上我是想设计一个class，所有的函数都隶属这个class来管理，并不想设计全局函数，那么错误在哪里呢

## 二. 报错信息的代码搜索

我做的第一件事情是，搜一下这个错误是在哪里报的错, php版本7.0.32， 在zend_API.c:2312行出现了这个报错

```c
    if (unload) { /* before unloading, display all remaining bad function in the module */
		if (scope) {
			efree((char*)lc_class_name);
		}
		while (ptr->fname) {
			fname_len = strlen(ptr->fname);
			lowercase_name = zend_string_alloc(fname_len, 0);
			zend_str_tolower_copy(ZSTR_VAL(lowercase_name), ptr->fname, fname_len);
			if (zend_hash_exists(target_function_table, lowercase_name)) {
				zend_error(error_type, "Function registration failed - duplicate name - %s%s%s", scope ? ZSTR_VAL(scope->name) : "", scope ? "::" : "", ptr->fname);
			}
			zend_string_free(lowercase_name);
			ptr++;
		}
		zend_unregister_functions(functions, count, target_function_table);
		return FAILURE;
	}
```
看一下调用栈，

```bash
gdb$ bt
#0  do_register_internal_class (orig_class_entry=0x7fffffffdf30, ce_flags=0x0) at /home/work/php-src-PHP-7.0.32/Zend/zend_API.c:2658
#1  0x0000000000a0448e in zend_register_internal_class (orig_class_entry=0x7fffffffdf30) at /home/work/php-src-PHP-7.0.32/Zend/zend_API.c:2707
#2  0x0000000000a13127 in zm_startup_core (type=0x1, module_number=0x0) at /home/work/php-src-PHP-7.0.32/Zend/zend_builtin_functions.c:354
#3  0x0000000000a01783 in zend_startup_module_ex (module=0x14501c0) at /home/work/php-src-PHP-7.0.32/Zend/zend_API.c:1847
#4  0x0000000000a017e5 in zend_startup_module_zval (zv=0x144cf50) at /home/work/php-src-PHP-7.0.32/Zend/zend_API.c:1862
#5  0x0000000000a0f23e in zend_hash_apply (ht=0x14435c0 <module_registry>, apply_func=0xa017c2 <zend_startup_module_zval>) at /home/work/php-src-PHP-7.0.32/Zend/zend_hash.c:1537
#6  0x0000000000a01daa in zend_startup_modules () at /home/work/php-src-PHP-7.0.32/Zend/zend_API.c:1973
#7  0x000000000096aff9 in php_module_startup (sf=0x141d620 <cli_sapi_module>, additional_modules=0x0, num_additional_modules=0x0) at /home/work/php-src-PHP-7.0.32/main/main.c:2239
#8  0x0000000000ab9537 in php_cli_startup (sapi_module=0x141d620 <cli_sapi_module>) at /home/work/php-src-PHP-7.0.32/sapi/cli/php_cli.c:426
#9  0x0000000000abb581 in main (argc=0x2, argv=0x1446830) at /home/work/php-src-PHP-7.0.32/sapi/cli/php_cli.c:1327
```
