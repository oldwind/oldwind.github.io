---
layout: content
title: mysql源码编译
status: 2
category: mysql, c
author:   "yimuren"
tags:
    - c
---

## 一. 问题描述

```c

cmake \
-DCMAKE_INSTALL_PREFIX=/home/work/source-learning/mysql/mysql-server/output \
-DMYSQL_DATADIR=/home/work/source-learning/mysql/mysql-server/output/data \
-DMYSQL_UNIX_ADDR=/home/work/source-learning/mysql/mysql-server/output/tmp/mysql.sock \
-DDEFAULT_CHARSET=utf8 \
-DDEFAULT_COLLATION=utf8_general_ci \
-DEXTRA_CHARSETS=all \
-DENABLED_LOCAL_INFILE=1 \
-DDOWNLOAD_BOOST=1 \
-DWITH_BOOST=/usr/local/boost \
-DWITH_DEBUG=1 \
-DCURSES_LIBRARY=/usr/lib/libncurses.so \
-DCURSES_INCLUDE_PATH=/usr/include

```
