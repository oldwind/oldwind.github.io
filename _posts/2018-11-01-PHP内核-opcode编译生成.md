---
layout: content
title: php内核opcode编译生成
status: 2 
category: c and c++
author:   "yimuren"
tags:
    - php
    - c
    - c++
---

## 一. 前言







## 二

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
filename:       /work/git-test/baidu/personal-code/yebin02-test/tmp/test_gc.php
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
filename:       /Users/baidu/work/git-test/baidu/personal-code/yebin02-test/tmp/test_gc.php
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





计划中

![opcodes结构图]({{site.baseurl}}/img/php/php5.4-opcodes.jpg)