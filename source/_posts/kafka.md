---
title: kafka
date: 2016-6-6 13:23:36
tags: [kafka,apache]
---

## kafka简介
### introduction
kafka: apache 旗下的一个项目

项目地址：[apache kafka](http://kafka.apache.org/)

apache kafka : a high-throughput distributed messaging system.

一个高吞吐量的分布式消息系统。具有四个特性：

- fast (快速)
- scalable (可伸缩)
- durable (可持久化)
- Distributed by Design (分布式设计)

### download & quickstart
1. download

下载地址： [kafka](http://kafka.apache.org/downloads.html)

2. quickstart

相关文档： [docs](http://kafka.apache.org/documentation.html)

**官方提供的脚本有bug,在 windows 环境下执行脚本会出错，原因是路径问题，为了避免出现问题，kafka的安装路径中不能出现空格。不要将其放置在：C:\\Program Files这样的路径下。**

[javadocs 0.8.2](https://kafka.apache.org/082/javadoc/)

[javadocs 0.9.0](https://kafka.apache.org/090/javadoc/)

[wiki](https://cwiki.apache.org/confluence/display/KAFKA/Index)

### kafka & zookeeper
kafka 在底层实现时使用了 zookeeper, 它们是如何协作的：

参考 ：

[What-is-the-actual-role-of-ZooKeeper-in-Kafka](https://www.quora.com/What-is-the-actual-role-of-ZooKeeper-in-Kafka)

[apache kafka系列之在zookeeper中存储结构](http://blog.csdn.net/lizhitao/article/details/23744675)

[kafka系列](http://blog.csdn.net/lizhitao/article/category/2194509)

3. kafka client api

kafka 提供的三种 client api.

* older scala client 有 两种： kafka.consumer 和 kafka.producer 是一种，还有一种，kafka.javaapi 这个包下的。
* org.apache.kafka.clients 基于 java 的包

## kafka相关

[linkedin kafka blog](https://engineering.linkedin.com/blog/topic/kafka)