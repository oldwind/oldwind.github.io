---
layout: content
title: php内核opcode编译生成
status: 1 
category: c and c++
author:   "yimuren"
tags:
    - php
    - c
    - c++
---

## 一. 前言

php的zend虚拟机在执行一个php的脚本过程中，经历了一系列"标准化"的流程，简单来说，包括词法分析、语法分析、生成opcode、执行opcode，关于对zend的opcode分析的文章也是不少，这里做了几件事
1. 学习了各种关于opcode的文章
2. 安装opcode分析的php扩展
3. 在mac上通过lldb来调试

核心的想法是把opcode的执行流程梳理一下，以及对opcode的核心指令做一下分析

## 二. vld工具安装

实际上，我们在调试过程中，可以打印出opcode，但是具体的含义还是不清楚的，例如下面通过lldb打印的一条指令，opcode是一条枚举指令，用"\0"表示，"\0"是什么，可读性比较差，zend的vm为了可读性，设计了一个数组，数组的索引是 opcode，具体含义用 字符串做了标示， vld可以清晰的打印出这个字符串，增强了可读性，所以这里我们可以下一下vld扩展，帮助我们去了解opcode

```bash
(lldb) p op_array->opcodes[0]
(zend_op) $50 = {
  handler = 0x0000000100729800
  op1 = (constant = 0, var = 0, num = 0, opline_num = 0, jmp_offset = 0)
  op2 = (constant = 0, var = 0, num = 0, opline_num = 0, jmp_offset = 0)
  result = (constant = 0, var = 0, num = 0, opline_num = 0, jmp_offset = 0)
  extended_value = 0
  lineno = 2
  opcode = '\0'
  op1_type = '\b'
  op2_type = '\b'
  result_type = '\b'
}

onst char *zend_vm_opcodes_map[173] = { }
```

### 2.1 安装过程

> 下载地址：https://github.com/derickr/vld/  
> git clone https://github.com/derickr/vld.git  
> cd vld  
> phpize  
> ./configure  
> make && make install  

修改一下php.ini，将扩展加载进去 后面通过 `php -m` 查看一下


### 2.2 执行

```bash
~/work/develope/php-7.0.29/bin/php -dvld.active=1  ./test_gc.php  
```

### 2.3 vld下opcode的阅读

我们先看一下用vld查看opcode的简单栗子，


{% highlight c %}
line     #* E I O op                           fetch          ext  return  operands
-------------------------------------------------------------------------------------
   2     0  E >   NOP                                                      
   7     1        INIT_FCALL                                               'test'
         2        DO_FCALL                                      0  $3      
         3        ASSIGN                                                   !0, $3
   8     4        IS_EQUAL                                         ~5      !0, 2
         5      > JMPZ                                                     ~5, ->7
{% endhighlight %}


标题的含义大体如下
- `line`, 写的php的的代码行号，
- `#*`， opcode的顺序
- `E`
- `I`
- `O`
- `op`， 操作码的名称，在zend_vm_codes.c中定义了一个map关系 zend_vm_opcodes_map
- `fetch`
- `ext`
- `return`
- `operands`

对于任意一个变量，zend虚拟机中都会重新修改变量的名字，vld在展示的时候，用符号做了一下区分，由于没有相关文档说明，我们翻一下vld的代码，可以看到，其中
- IS_UNUSED           对于定义了，但是没有使用的变量，vld不展示
- IS_CONST            对于常量，vld，直接展示长量值
- IS_TMP_VAR    `～`  标示变量是一个临时变量，例如在一个fuction内部定义的变量
- IS_VAR        `$`   
- IS_CV         `!`   Compiled variable, 编译中产生的中间变量，类似缓存，例如 function的参数，会生成中间变量存储调用过程中传递的参数值
- VLD_IS_OPNUM  `->`
- VLD_IS_OPLINE `->`
- VLD_IS_CLASS  `:`   标示操作数是一个class变量  

```c
int vld_dump_znode (int *print_sep, unsigned int node_type, VLD_ZNODE node, unsigned int base_address, zend_op_array *op_array, int opline TSRMLS_DC)
{
    ...
    switch (node_type) {
        case IS_UNUSED:
            VLD_PRINT(3, " IS_UNUSED ");
            break;
        case IS_CONST: /* 1 */
            VLD_PRINT1(3, " IS_CONST (%d) ", VLD_ZNODE_ELEM(node, var) / sizeof(zval));
...
            break;

        case IS_TMP_VAR: /* 2 */
            VLD_PRINT(3, " IS_TMP_VAR ");
            len += vld_printf (stderr, "~%d", VAR_NUM(VLD_ZNODE_ELEM(node, var)));
            break;
        case IS_VAR: /* 4 */
            VLD_PRINT(3, " IS_VAR ");
            len += vld_printf (stderr, "$%d", VAR_NUM(VLD_ZNODE_ELEM(node, var)));
            break;
        case IS_CV:  /* 16 */
            VLD_PRINT(3, " IS_CV ");
            len += vld_printf (stderr, "!%d", (VLD_ZNODE_ELEM(node, var)-sizeof(zend_execute_data)) / sizeof(zval));
            break;
        case VLD_IS_OPNUM:
            len += vld_printf (stderr, "->%d", VLD_ZNODE_JMP_LINE(node, opline, base_address));
            break;
        case VLD_IS_OPLINE:
            len += vld_printf (stderr, "->%d", VLD_ZNODE_JMP_LINE(node, opline, base_address));
            break;
        case VLD_IS_CLASS:
            len += vld_printf (stderr, ":%d", VAR_NUM(VLD_ZNODE_ELEM(node, var)));
            break;
...
        default:
            return 0;
    }
    return len;
}


```




## 三. 核心数据结构分析

这里先说一下，选择的php的版本是7.0.29，

```c
struct _zend_compiler_globals {
    zend_stack loop_var_stack;

    zend_class_entry *active_class_entry;

    zend_string *compiled_filename;

    int zend_lineno;

    zend_op_array *active_op_array;

    HashTable *function_table;    /* function symbol table */
    HashTable *class_table;        /* class table */

    HashTable filenames_table;

    HashTable *auto_globals;

    zend_bool parse_error;
    zend_bool in_compilation;
    zend_bool short_tags;

    zend_bool unclean_shutdown;

    zend_bool ini_parser_unbuffered_errors;

    zend_llist open_files;

    struct _zend_ini_parser_param *ini_parser_param;

    uint32_t start_lineno;
    zend_bool increment_lineno;

    zend_string *doc_comment;

    uint32_t compiler_options; /* set of ZEND_COMPILE_* constants */

    HashTable const_filenames;

    zend_oparray_context context;
    zend_file_context file_context;

    zend_arena *arena;

    zend_string *empty_string;
    zend_string *one_char_string[256];

    HashTable interned_strings;

    const zend_encoding **script_encoding_list;
    size_t script_encoding_list_size;
    zend_bool multibyte;
    zend_bool detect_unicode;
    zend_bool encoding_declared;

    zend_ast *ast;
    zend_arena *ast_arena;

    zend_stack delayed_oplines_stack;

#ifdef ZTS
    zval **static_members_table;
    int last_static_member;
#endif
};


struct _zend_executor_globals {
    zval uninitialized_zval;
    zval error_zval;

    /* symbol table cache */
    zend_array *symtable_cache[SYMTABLE_CACHE_SIZE];
    zend_array **symtable_cache_limit;
    zend_array **symtable_cache_ptr;

    zend_array symbol_table;        /* main symbol table */

    HashTable included_files;    /* files already included */

    JMP_BUF *bailout;

    int error_reporting;
    int exit_status;

    HashTable *function_table;    /* function symbol table */
    HashTable *class_table;        /* class table */
    HashTable *zend_constants;    /* constants table */

    zval          *vm_stack_top;
    zval          *vm_stack_end;
    zend_vm_stack  vm_stack;

    struct _zend_execute_data *current_execute_data;
    zend_class_entry *scope;

    zend_long precision;

    int ticks_count;

    HashTable *in_autoload;
    zend_function *autoload_func;
    zend_bool full_tables_cleanup;

    /* for extended information support */
    zend_bool no_extensions;

#ifdef ZEND_WIN32
    zend_bool timed_out;
    OSVERSIONINFOEX windows_version_info;
#endif

    HashTable regular_list;
    HashTable persistent_list;

    int user_error_handler_error_reporting;
    zval user_error_handler;
    zval user_exception_handler;
    zend_stack user_error_handlers_error_reporting;
    zend_stack user_error_handlers;
    zend_stack user_exception_handlers;

    zend_error_handling_t  error_handling;
    zend_class_entry      *exception_class;

    /* timeout support */
    zend_long timeout_seconds;

    int lambda_count;

    HashTable *ini_directives;
    HashTable *modified_ini_directives;
    zend_ini_entry *error_reporting_ini_entry;

    zend_objects_store objects_store;
    zend_object *exception, *prev_exception;
    const zend_op *opline_before_exception;
    zend_op exception_op[3];

    struct _zend_module_entry *current_module;

    zend_bool active;
    zend_bool valid_symbol_table;

    zend_long assertions;

    uint32_t           ht_iterators_count;     /* number of allocatd slots */
    uint32_t           ht_iterators_used;      /* number of used slots */
    HashTableIterator *ht_iterators;
    HashTableIterator  ht_iterators_slots[16];

    void *saved_fpu_cw_ptr;
#if XPFPA_HAVE_CW
    XPFPA_CW_DATATYPE saved_fpu_cw;
#endif

    zend_function trampoline;
    zend_op       call_trampoline_op;

    void *reserved[ZEND_MAX_RESERVED_RESOURCES];
};

struct _zend_op {
    const void *handler;
    znode_op op1;   // 操作数1
    znode_op op2;   // 操作数2
    znode_op result;  // 操作结果
    uint32_t extended_value;
    uint32_t lineno;
    zend_uchar opcode;  
    zend_uchar op1_type;  // 操作数1的类型
    zend_uchar op2_type;   // 操作数2的类型
    zend_uchar result_type;   // 操作结果类型
};

typedef union _znode_op {
    uint32_t      constant;
    uint32_t      var;
    uint32_t      num;
    uint32_t      opline_num; /*  Needs to be signed */
#if ZEND_USE_ABS_JMP_ADDR
    zend_op       *jmp_addr;
#else
    uint32_t      jmp_offset;
#endif
#if ZEND_USE_ABS_CONST_ADDR
    zval          *zv;
#endif
} znode_op;


struct _zend_execute_data {
    const zend_op       *opline;   /* executed opline                */
    zend_execute_data   *call;     /* current call                   */
    zval                *return_value;
    zend_function       *func;     /* executed funcrion              */
    zval                 This;     /* this + call_info + num_args    */
    zend_class_entry    *called_scope;
    zend_execute_data   *prev_execute_data;
    zend_array          *symbol_table;
#if ZEND_EX_USE_RUN_TIME_CACHE
    void               **run_time_cache; /* cache op_array->run_time_cache */
#endif
#if ZEND_EX_USE_LITERALS
    zval                *literals;  /* cache op_array->literals       */
#endif
};

```




![opcodes结构图]({{site.baseurl}}/img/php/php5.4-opcodes.jpg)




## 四. 数据流程




```bash
(lldb) bt
* thread #1, queue = 'com.apple.main-thread', stop reason = step over
  * frame #0: 0x00000001006625c3 php`compile_file(file_handle=0x00007ffeefbff098, type=8) at zend_language_scanner.l:626
    frame #1: 0x00000001003a70ee php`phar_compile_file(file_handle=0x00007ffeefbff098, type=8) at phar.c:3337
    frame #2: 0x00000001006c564a php`zend_execute_scripts(type=8, retval=0x0000000000000000, file_count=3) at zend.c:1439
    frame #3: 0x000000010061bdf1 php`php_execute_script(primary_file=0x00007ffeefbff098) at main.c:2516
    frame #4: 0x00000001007b56fd php`do_cli(argc=2, argv=0x00007ffeefbff7a0) at php_cli.c:977
    frame #5: 0x00000001007b4691 php`main(argc=2, argv=0x00007ffeefbff7a0) at php_cli.c:1347
    frame #6: 0x00007fff59379085 libdyld.dylib`start + 1
(lldb)
```

### 4.1 数据的编译分析

```php
<?php 
function test() {
    echo "it's test";
    return 1;
}

$b = test();
if ($b == 2) {
    echo "it is not right";
}

class Mtest {
    public $i = 100;
}

$obj = new Mtest();
var_dump($obj->i);

unset($obj);
for ($i = 0; $i < 10; $i ++ ) {
    echo "ssss\n";
}
```


{% highlight c %}
Finding entry points
Branch analysis from position: 0
2 jumps found. (Code = 43) Position 1 = 6, Position 2 = 7
Branch analysis from position: 6
1 jumps found. (Code = 42) Position 1 = 21
Branch analysis from position: 21
2 jumps found. (Code = 44) Position 1 = 23, Position 2 = 18
Branch analysis from position: 23
1 jumps found. (Code = 62) Position 1 = -2
Branch analysis from position: 18
2 jumps found. (Code = 44) Position 1 = 23, Position 2 = 18
Branch analysis from position: 23
Branch analysis from position: 18
Branch analysis from position: 7
filename:           /personal-code/yebin02-test/tmp/test_gc.php
function name:  (null)
number of ops:  24
compiled vars:  !0 = $b, !1 = $obj, !2 = $i
line     #* E I O op                           fetch          ext  return  operands
-------------------------------------------------------------------------------------
   2     0  E >   NOP                                                      
   7     1        INIT_FCALL                                               'test'
         2        DO_FCALL                                      0  $3      
         3        ASSIGN                                                   !0, $3
   8     4        IS_EQUAL                                         ~5      !0, 2
         5      > JMPZ                                                     ~5, ->7
   9     6    >   ECHO                                                     'it+is+not+right'
  12     7    >   NOP                                                      
  17     8        NEW                                              $7      :-3
         9        DO_FCALL                                      0          
        10        ASSIGN                                                   !1, $7
  18    11        INIT_FCALL                                               'var_dump'
        12        FETCH_OBJ_R                                      $10     !1, 'i'
        13        SEND_VAR                                                 $10
        14        DO_ICALL                                                 
  20    15        UNSET_VAR                                                !1
  21    16        ASSIGN                                                   !2, 0
        17      > JMP                                                      ->21
  22    18    >   ECHO                                                     'ssss%0A'
  21    19        POST_INC                                         ~13     !2
        20        FREE                                                     ~13
        21    >   IS_SMALLER                                       ~14     !2, 10
        22      > JMPNZ                                                    ~14, ->18
  24    23    > > RETURN                                                   1

branch: #  0; line:     2-    8; sop:     0; eop:     5; out0:   6; out1:   7
branch: #  6; line:     9-   12; sop:     6; eop:     6; out0:   7
branch: #  7; line:    12-   21; sop:     7; eop:    17; out0:  21
branch: # 18; line:    22-   21; sop:    18; eop:    20; out0:  21
branch: # 21; line:    21-   21; sop:    21; eop:    22; out0:  23; out1:  18; out2:  23; out3:  18
branch: # 23; line:    24-   24; sop:    23; eop:    23; out0:  -2
path #1: 0, 6, 7, 21, 23, 
path #2: 0, 6, 7, 21, 18, 21, 23, 
path #3: 0, 6, 7, 21, 18, 21, 23, 
path #4: 0, 6, 7, 21, 23, 
path #5: 0, 6, 7, 21, 18, 21, 23, 
path #6: 0, 6, 7, 21, 18, 21, 23, 
path #7: 0, 7, 21, 23, 
path #8: 0, 7, 21, 18, 21, 23, 
path #9: 0, 7, 21, 18, 21, 23, 
path #10: 0, 7, 21, 23, 
path #11: 0, 7, 21, 18, 21, 23, 
path #12: 0, 7, 21, 18, 21, 23, 
Function test:
Finding entry points
Branch analysis from position: 0
1 jumps found. (Code = 62) Position 1 = -2
filename:               /personal-code/yebin02-test/tmp/test_gc.php
function name:  test
number of ops:  3
compiled vars:  none
line     #* E I O op                           fetch          ext  return  operands
-------------------------------------------------------------------------------------
   3     0  E >   ECHO                                                     'it%27s+test'
   4     1      > RETURN                                                   1
   5     2*     > RETURN                                                   null

branch: #  0; line:     3-    5; sop:     0; eop:     2
path #1: 0, 
End of function test

Class Mtest: [no user functions]
{% endhighlight %}


### 4.2 opcode代码的执行



```c
ZEND_API void execute_ex(zend_execute_data *ex)
{
    DCL_OPLINE

#ifdef ZEND_VM_IP_GLOBAL_REG
    const zend_op *orig_opline = opline;
#endif
#ifdef ZEND_VM_FP_GLOBAL_REG
    zend_execute_data *orig_execute_data = execute_data;
    execute_data = ex;
#else
    zend_execute_data *execute_data = ex;
#endif
    LOAD_OPLINE();

    while (1) {
#if !defined(ZEND_VM_FP_GLOBAL_REG) || !defined(ZEND_VM_IP_GLOBAL_REG)
        int ret;
#endif
#if defined(ZEND_VM_FP_GLOBAL_REG) && defined(ZEND_VM_IP_GLOBAL_REG)
        ((opcode_handler_t)OPLINE->handler)(ZEND_OPCODE_HANDLER_ARGS_PASSTHRU);
        if (UNEXPECTED(!OPLINE)) {
#else
        if (UNEXPECTED((ret = ((opcode_handler_t)OPLINE->handler)(ZEND_OPCODE_HANDLER_ARGS_PASSTHRU)) != 0)) {
#endif
#ifdef ZEND_VM_FP_GLOBAL_REG
            execute_data = orig_execute_data;
# ifdef ZEND_VM_IP_GLOBAL_REG
            opline = orig_opline;
# endif
            return;
#else
            if (EXPECTED(ret > 0)) {
                execute_data = EG(current_execute_data);
            } else {
# ifdef ZEND_VM_IP_GLOBAL_REG
                opline = orig_opline;
# endif
                return;
            }
#endif
        }
    }
    zend_error_noreturn(E_CORE_ERROR, "Arrived at end of main loop which shouldn't happen");
}
```

我们看一下

{% highlight c %}
(lldb) bt
* thread #1, queue = 'com.apple.main-thread', stop reason = breakpoint 9.1
  * frame #0: 0x0000000100729634 php`execute_ex(ex=0x0000000102019030) at zend_vm_execute.h:417
    frame #1: 0x000000010072979a php`zend_execute(op_array=0x000000010206d400, return_value=0x0000000000000000) at zend_vm_execute.h:458
    frame #2: 0x00000001006c56a2 php`zend_execute_scripts(type=8, retval=0x0000000000000000, file_count=3) at zend.c:1445
    frame #3: 0x000000010061bdf1 php`php_execute_script(primary_file=0x00007ffeefbff098) at main.c:2516
    frame #4: 0x00000001007b56fd php`do_cli(argc=2, argv=0x00007ffeefbff7a0) at php_cli.c:977
    frame #5: 0x00000001007b4691 php`main(argc=2, argv=0x00007ffeefbff7a0) at php_cli.c:1347
    frame #6: 0x00007fff59379085 libdyld.dylib`start + 1
{% endhighlight %}