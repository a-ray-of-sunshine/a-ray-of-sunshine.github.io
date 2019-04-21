---
title: redis-源码分析
date: 2018-3-24 14:27:23
---

## redis 源码目录结构

redis源码主要分为下面几大类

### 数据结构

#### adlist.c

A generic doubly linked list implementation

#### sds.c

 A C dynamic strings library

#### ae*.c

A simple event-driven programming library.

#### anet.c

Basic TCP socket stuff made a bit less boring

封闭了TCP socket 常用函数，例如 Socket 的 create, accept, write, read

#### AOF

* aof.c
* bio.c

	Background I/O service for Redis.

* rio.c
	
	rio.c is a simple stream-oriented I/O abstraction that provides an interface to write code that can consume/produce data using different concrete input and output devices.

#### bitops.c

Bit operations

#### blocked.c 

generic support for blocking operations like BLPOP & WAIT.

#### cluster.c

Redis Cluster implementation.

#### config.c

Configuration file parsing and CONFIG GET/SET commands implementation.

#### CRC

crc16.c crc64.c

#### Hash Table

dict.c

#### LZF 压缩算法

### 命令（redis command）实现相关

#### db.c

C-level DB API

#### expire.c

Implementation of EXPIRE (keys with fixed time to live).

#### geo.c

### 内存操作相关

#### evict.c 

Maxmemory directive handling (LRU eviction and other policies).

#### defrag.c

### Redis Module

module.c

### redis 对象

object.c

### replication

Asynchronous replication implementation.




## $.参考
1. []()
2. []()
