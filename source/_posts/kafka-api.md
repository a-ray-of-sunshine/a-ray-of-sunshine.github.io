---
title: kafka-api
date: 2016-6-6 13:23:36
tags: [kafka,api]
---

## kafka api
kafka 提供的 client api 共有3种，这三种也是随着 kafka 版本升级不断演进的。
前两种是 使用 scala 编写的api,
在 kafka 

### 0.7 版本的api

可以参考文档： [0.7文档](http://kafka.apache.org/07/quickstart.html)

在这个版本中将 client api 也就是 （consumer 和 producer api 分为两类）， high-level 和 lower-level

####　high-level

包含在 ： kafka.producer 和 kafka.consumer 这两个包下面，这 high-level 是基于 zookeeper 来实现的。

#### lower-level

Kafka has a lower-level consumer api for reading message chunks directly from servers. Under most circumstances this should not be needed.

这个 lower-level api 主要指的就是 
kafka.javaapi.consumer.SimpleConsumer 这个 consumer api

查看 源码 可知 这个 consumer 的低层实现 依赖于 
kafka.consumer.SimpleConsumer， 而它的实现是不依赖 zookeeper的 .

### 0.8.0 / 0.8.1 版本的api

[0.8.0文档](http://kafka.apache.org/08/documentation.html)

介绍的 Producer API 是
kafka.javaapi.producer.Producer 这个类，在包kafka.javaapi.producer中，其实个包和类在 0.7 版本中就已经存在了。

#### high-level
还是指：
kafka.producer 和 kafka.consumer 

#### lower-level

kafka.javaapi.consumer.SimpleConsumer 

For most applications, the high level consumer Api is good enough. Some applications want features not exposed to the high level consumer yet (e.g., set initial offset when restarting the consumer). They can instead use our low level SimpleConsumer Api.

### 0.8.2 版本的api

We are in the process of rewritting the JVM clients for Kafka. As of 0.8.2 Kafka includes a newly rewritten Java producer. The next release will include an equivalent Java consumer. These new clients are meant to supplant the existing Scala clients, but for compatability they will co-exist for some time. These clients are available in a seperate jar with minimal dependencies, while the old Scala clients remain packaged with the server.

摘自文档 [0.8.2 文档](http://kafka.apache.org/082/documentation.html)

As of the 0.8.2 release we encourage all new development to use the new Java producer. This client is production tested and generally both faster and more fully featured than the previous Scala client. You can use this client by adding a dependency on the client jar using the following maven co-ordinates:

	<dependency>
	    <groupId>org.apache.kafka</groupId>
	    <artifactId>kafka-clients</artifactId>
	    <version>0.8.2.0</version>
	</dependency>
	
在这个版本中 官方推荐：对于 producer 可以使用新的 java编写的 kafka-clients 这个包的功能，而 consumer 并没有提到使用新的 api ,此时 consumer 仍然使用上面的 high-level 和 low-level api. 

### 0.9.x 版本的api

参考文档： [0.9.0 文档](http://kafka.apache.org/090/documentation.html)

文档中描述： producer api 推荐使用基于java的包。也就是0.8.2中提到的 org.apache.kafka 这个包。

**摘自官方文档：**

As of the 0.9.0 release we have added a new Java consumer to replace our existing high-level ZooKeeper-based consumer and low-level consumer APIs. This client is considered beta quality. To ensure a smooth upgrade path for users, we still maintain the old 0.8 consumer clients that continue to work on an 0.9 Kafka cluster. In the following sections we introduce both the old 0.8 consumer APIs (both high-level ConsumerConnector and low-level SimpleConsumer) and the new Java consumer API respectively.

其中：

* high-level ZooKeeper-based consumer： 就是 kafka.consumer 这个包中的文件。

* low-level consumer APIs： 就是kafka.consumer.SimpleConsumer 

* This client is considered beta quality：这个版本的consumer 还是处于 beta quality。同时之前的api都是可以正常使用的。

### 0.10.x 版本的api

参考文档： [0.10.0](http://kafka.apache.org/documentation.html)

依照文档描述，这个版本中的 client-api 仍处于 beta quality。

在这个版本中又引入了一个新的：

**Streams API**

As of the 0.10.0 release we have added a new client library named Kafka Streams to let users implement their stream processing applications with data stored in Kafka topics. Kafka Streams is considered alpha quality and its public APIs are likely to change in future releases. You can use Kafka Streams by adding a dependency on the streams jar using the following example maven co-ordinates (you can change the version numbers with new releases):

	<dependency>
	    <groupId>org.apache.kafka</groupId>
	    <artifactId>kafka-streams</artifactId>
	    <version>0.10.0.0</version>
	</dependency>
	
Examples showing how to use this library are given in the javadocs (note those classes annotated with @InterfaceStability.Unstable, indicating their public APIs may change without backward-compatibility in future releases).

## api 使用示例

### 0.7.x 

使用示例：[示例](http://kafka.apache.org/07/quickstart.html)

这个版本中使用的一些类，已经在0.8以后的版本中不存在了。

### 0.8.0 / 0.8.1 

#### high-level

* producer api

使用示例：
[Producer Example](https://cwiki.apache.org/confluence/display/KAFKA/0.8.0+Producer+Example)

* consumer api
使用示例：
[Consumer Group Example](https://cwiki.apache.org/confluence/display/KAFKA/Consumer+Group+Example)

#### lower-level

* SimpleConsumer
[SimpleConsumer Example](https://cwiki.apache.org/confluence/display/KAFKA/0.8.0+SimpleConsumer+Example)

### 0.8.2

引入新的 client-api  org.apache.kafka包。

其中新的 Producer API 使用方法：
[KafkaProducer](http://kafka.apache.org/082/javadoc/index.html?org/apache/kafka/clients/producer/KafkaProducer.html)

### 0.9.0

引入新的 client-api  org.apache.kafka包。

[KafkaConsumer](http://kafka.apache.org/090/javadoc/index.html?org/apache/kafka/clients/consumer/KafkaConsumer.html)

### 0.10.0

* [KafkaProducer](http://kafka.apache.org/0100/javadoc/index.html?org/apache/kafka/clients/producer/KafkaProducer.html)

* [KafkaConsumer](http://kafka.apache.org/0100/javadoc/index.html?org/apache/kafka/clients/consumer/KafkaConsumer.html)