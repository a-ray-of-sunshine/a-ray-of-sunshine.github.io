---
title: pinpoint-源码分析
date: 2016-12-8 15:54:44
---

## web

### 项目配置

主要的配置文件：

* hbase.properties
* jdbc.properties
* pinpoint-web.properties

这些配置文件在 applicationContext-web.xml 文件中被注入成 bean

``` xml
<bean id="propertyConfigurer" class="org.springframework.beans.factory.config.PropertyPlaceholderConfigurer">
    <property name="locations">
        <list>
            <value>classpath:hbase.properties</value>
            <value>classpath:jdbc.properties</value>
        </list>
    </property>
</bean>

<bean id="pinpointWebProps" class="org.springframework.beans.factory.config.PropertiesFactoryBean">
    <property name="location" value="classpath:pinpoint-web.properties"/>
    <property name="fileEncoding" value="UTF-8"/>
</bean>
```

在项目使用 @Value 注解的方式，使用上面的配置文件。

主要的配置类在 `com.navercorp.pinpoint.web.config` 包中。

### 集群

pinpoint web 端自身可以集群。

`com.navercorp.pinpoint.web.cluster` 是和集群相关的包

集群的入口 `com.navercorp.pinpoint.web.cluster.ClusterManager`

### DAO 层的实现

数据存储使用 hbase

数据读取使用 org.springframework.data.hadoop.hbase.RowMapper 创建一个对应的 Mapper 类。 在 `com.navercorp.pinpoint.web.mapper`

DAO 层使用这些 mapper 对象，进行数据库的读写。

## collector

collector 自身也可以做集群

### collector 集群

集群相关的包 `com.navercorp.pinpoint.collector.cluster`

集群的入口 `com.navercorp.pinpoint.collector.cluster.zookeeper.ZookeeperClusterService`

## agent

agent 实现的核心是 **pinpoint-bootstrap** 和 **pinpoint-bootstrap-core** 两个项目。

其中 **pinpoint-bootstrap** 项目，作为 agent 的入口，其主要作用就是加载 **pinpoint-bootstrap-core**。

所以 **pinpoint-bootstrap-core** 是整个 agent 的核心实现。

**pinpoint-agent** 项目中的 `pinpoint.config` 文件是整个插件存储的配置信息。

其中的 `profiler.collector.tcp.ip` 参数，用来配置 collector 的 ip。

所以插件将信息交给 collector, 由 collector 进行存储。

下面分别分析和 Agent 相关的项目

### pinpoint-bootstrap

使用是加载 agent.

### pinpoint-bootstrap-core

这个项目提供了分布式追踪的抽象模型，大多都是接口。

* 初始化 Agent 相关的配置

	代码分布在 `com.navercorp.pinpoint.bootstrap.config` 包中

* 提供实现分布式追踪的抽象模型

	`com.navercorp.pinpoint.bootstrap.context` 提供了分布式追踪的抽象接口。
	
* 插件模型

	`com.navercorp.pinpoint.bootstrap.plugin`

* 字节码注入技术

	pinpoint 将字节码注入进行了进一步的抽象，将其抽象为拦截器。

### pinpoint-profiler

agent 实现的分布式请求追踪功能的相关数据结构就在这个项目中定义。

实现 pinpoint-bootstrap-core 这个项目中的接口。

### pinpoint-profiler-optional
### pinpoint-rpc
### pinpoint-thrift
### pinpoint-xxx-plugin


## 其它实现

基于 Google Dapper 的分布式实时数据追踪系统

1. [Zipkin is a distributed tracing system](https://github.com/openzipkin/zipkin)
2. [brave - Java distributed tracing implementation compatible with Zipkin back-end services.](https://github.com/openzipkin/brave)

## mysql 插件的实现

### sql 语句的跟踪

* PreparedStatement

	由 PreparedStatementCreateInterceptor ，通过拦截 Connection 的 prepareStatement 方法来实现，
	
* Statement

	由 StatementExecuteQueryInterceptor 和 StatementExecuteUpdateInterceptor 接口对，executeUpdate， execute， executeQuery 方法进行拦截。


## 参考
1. [使用Spring 3的@value简化配置文件的读取](http://www.cnblogs.com/BensonHe/p/3963940.html)
2. [Dapper：谷歌的大规模分布式跟踪系统](http://blog.jobbole.com/69642/)
2. [Dapper, a Large-Scale Distributed Systems Tracing Infrastructure](https://github.com/bigbully/Dapper-translation/blob/master/dapper%E5%88%86%E5%B8%83%E5%BC%8F%E8%B7%9F%E8%B8%AA%E7%B3%BB%E7%BB%9F%E5%8E%9F%E6%96%87.pdf)
4. [分布式系统为什么需要 Tracing](http://www.cnblogs.com/zhengyun_ustc/p/55solution2.html)
