---
title: BigData-kafka
date: 2016-11-22 10:45:17
---

## 常用命令

``` bash
bin/kafka-run-class.sh kafka.admin.ConsumerGroupCommand
bin/kafka-run-class.sh kafka.tools.ConsumerOffsetChecker

bin/kafka-run-class.sh kafka.admin.ConsumerGroupCommand --describe --group dbanalysis-storm-reader  --zookeeper 192.168.0.117:2181

bin/kafka-run-class.sh kafka.tools.ConsumerOffsetChecker --group dbanalysis-storm-reader --topic OracleFantasy --zookeeper 192.168.0.117:2181

## 列出所有的 new consumer
bin/kafka-run-class.sh kafka.admin.ConsumerGroupCommand --list --new-consumer --bootstrap-server 192.168.0.20:9092

bin/kafka-run-class.sh kafka.admin.ConsumerGroupCommand --describe --new-consumer --group storm-reader --bootstrap-server 192.168.0.20:9092
```

## Kafka Server 的启动

kafka.server.KafkaConfig kafka配置

kafka.utils.ZkUtils zookeeper 配置

kafka.server.KafkaServer.startup() 启动 Kafka

### 创建调度器

kafka.server.KafkaServer.kafkaScheduler

``` scala
/* start scheduler */
// 默认有10个线程。
kafkaScheduler.startup()
```

线程的名称为 "kafka-scheduler-"

这个调度器，用来执行一些 kafka 常用任务。

### initZK

创建 ZkClient 对象。并创建以下路径

``` scala
/* setup zookeeper */
zkClient = initZk()
      
val ConsumersPath = "/consumers"
val BrokerIdsPath = "/brokers/ids"
val BrokerTopicsPath = "/brokers/topics"
val TopicConfigPath = "/config/topics"
val TopicConfigChangesPath = "/config/changes"
val DeleteTopicsPath = "/admin/delete_topics"
```

### 启动 LogManager

创建 kafka.log.LogManager 对象。并启动 logManager

``` scala
/* start log manager */
logManager = createLogManager(zkClient, brokerState)
logManager.startup()
```

* 向调度器中提交三个任务

	1. kafka-log-retention
	
		kafka.log.LogManager.cleanupLogs()
		
		Delete any eligible logs. Return the number of segments deleted.
		
	2. kafka-log-flusher
	
		kafka.log.LogManager.flushDirtyLogs()
		
		Flush any log which has exceeded its flush interval and has unwritten messages.
		
	3. kafka-recovery-point-checkpoint
	
		kafka.log.LogManager.checkpointRecoveryPointOffsets()
		
		Write out the current recovery point for all logs to a text file in the log directory to avoid recovering the whole log on startup.

* 创建 kafka-log-cleaner-thread 线程

	kafka.log.LogCleaner

### 创建Server

kafka.network.SocketServer

An NIO socket server. The threading model is 

1 Acceptor thread that handles new connections 

N Processor threads that each have their own selector and read requests from sockets 

M Handler threads that handle requests and produce responses back to the processor threads for writing.

* 创建 Processor 线程

	创建 num.network.threads 个 Processor 线程，默认是 3 个线程。线程名称 kafka-network-thread-%d-%d

* 启动 kafka.network.Acceptor

	启动一个名为 kafka-socket-acceptor 的线程。这个线程将在 host.name 和 port 上监听。 port 默认为 9092

### 创建 kafka.server.ReplicaManager

向 scheduler 添加一个定时任务。进行分区的复制任务。

``` scala
def startup() {
	// start ISR expiration thread
	scheduler.schedule("isr-expiration", maybeShrinkIsr, period = config.replicaLagTimeMaxMs, unit = TimeUnit.MILLISECONDS)
}
```

### 启动 kafka.server.OffsetManager

offset管理器。向 scheduler 添加一个定时任务。执行 offset cache 的压缩操作。

### kafka.server.KafkaApis

Logic to handle the various Kafka requests

处理 Producer 和 Consumer 读写， 元数据查询，offset 管理等请求。

### 请求处理池线程

``` scala
/* start processing requests */
apis = new KafkaApis(socketServer.requestChannel, replicaManager, consumerCoordinator,
  kafkaController, zkUtils, config.brokerId, config, metadataCache, metrics, authorizer)
requestHandlerPool = new KafkaRequestHandlerPool(config.brokerId, socketServer.requestChannel, apis, config.numIoThreads)
brokerState.newState(RunningAsBroker)
```

创建 num.io.threads 个线程（默认8个），使用 kafka.server.KafkaRequestHandler 处理请求。

KafkaRequestHandler 交给 kafka.server.KafkaApis 的 handle 方法进行处理。

kafka.network.RequestChannel.Request 解析请求。

## offset 管理

kafka 对于 offset 的管理。Kafka 在 0.8.1.1 这个版本之前，将 Consumer 消费的 offset 的保存到 zookeeper 中，从这个版本以后，kafka Server 会创建一个名称为 __consumer_offset 分区个数为 50 个 的 topic 来存储 consumer 消费的 offset. 0.8.1.1 以后的版本默认使用 后面的机制来存储 offset.

可以在 kafka server 的 config/server.properties 中配置 __consumer_offset 分区的数量，使用参数： offsets.topic.num.partitions，默认是50.

在 kafka 版本为 0.8.2.0 这个版本发布的时候，同时发布了一个 Java 版的 client. 这个新版本的 API 称为， New API。这个新版本API对于 offset 的管理，提供了两种方式，一种是自动周期性提交，一种是使用是 KafkaConsumer r的commit 系列 API 进行手动提交。

如果自动提交，可以使用 auto.commit.interval.ms 设置提交的时间间隔。如果设置 enable.auto.commit 为 false, 则不进行自动提交。如果不进行自动提交，有两种选择，可以使用 commit API 将 offset 交由 kafka 保存，也可以自行保存 offset, 例如将 offset 保存至 数据库。

Kafka 在 0.8.2.0 这个版本之前提供的 Consumer 和 Producer API 是在 Kafka 实现包中，是使用 scala 写的。这类API分 heigh-level 和 low-level API。这些 Old API 可以通过设置 offsets.storage 为 zookeeper 或 kafka. 来表示 offset 的存储位置。默认 Old API使用 zookeeper 进行存储。New Client API 默认使用 kafka 存储，不提供存储设置选项，只能使用 kafka 存储。

## kafka consumer API

* Low-level

	[SimpleConsumer](https://cwiki.apache.org/confluence/display/KAFKA/0.8.0+SimpleConsumer+Example)
	
	SimpleConsumer 提供了 commitOffsets 方法，可以用来提交 offset 到 kafka 进行管理。使用 getOffsetsBefore 方法可以进行 offset 的查询。
	
* High-level

	[Using the High Level Consumer](https://cwiki.apache.org/confluence/display/KAFKA/Consumer+Group+Example)
	
	核心实现：kafka.consumer.ZookeeperConsumerConnector, 其使用 kafka.consumer.ConsumerConfig 进行配置。
	
	可以使用 auto.commit.enable 配置来设置是否自动提交 offset. 默认是 true. 并且默认是存储到 zookeeper.
	
	也就是说使用 high-levle API, 默认每隔1分钟，将offset提交到 zookeepr 中。
	
	 /consumers/[group_id]/offsets/[topic]/[broker_id-partition_id] --> offset_counter_value
	
* New client API

	提供新的 jar 包, 从 kafka 0.8.2.0 这个版本开始提供，使用 java 实现。 
	
	[KafkaProducer](http://kafka.apache.org/090/javadoc/index.html?org/apache/kafka/clients/producer/KafkaProducer.html)
	
	[KafkaConsumer](http://kafka.apache.org/090/javadoc/index.html?org/apache/kafka/clients/consumer/KafkaConsumer.html)

## New Client API

New Client API分两种方式消费 topic. subscribe 和 assign 方法。

### assign 

直接分配给当前 Consumer 去消费 topic 的 partition.不会进行 load balancing.

### subscribe 

是只需要指定 topic, 至于 consumer 具体消费哪个 partition. 则由 Kafka Server 进行动态协调。这样属于同一个组的多个 Consumer 可以进行动态的 load balancing。这种方式也称为按组消费.

## kafka 实现 



## $.参考
1. [Kafka架构及HA实现分析](http://www.jasongj.com/categories/Kafka/)
2. [Committing and fetching consumer offsets in Kafka](https://cwiki.apache.org/confluence/display/KAFKA/Committing+and+fetching+consumer+offsets+in+Kafka)
3. [Consumer Offset Tracking](http://kafka.apache.org/082/documentation.html#distributionimpl)
4. [New Client API](http://kafka.apache.org/090/javadoc/index.html)
5. [System Tools](https://cwiki.apache.org/confluence/display/KAFKA/System+Tools)
6. [Kafka data structures in Zookeeper](https://cwiki.apache.org/confluence/display/KAFKA/Kafka+data+structures+in+Zookeeper)
7. [miguno/kafka-storm-starter](https://github.com/miguno/kafka-storm-starter)
8. [streams](http://kafka.apache.org/documentation/streams)
9. [Kafka Streams](http://docs.confluent.io/2.1.0-alpha1/streams/index.html)
10. [introducing-kafka-streams-stream-processing-made-simple](https://www.confluent.io/blog/introducing-kafka-streams-stream-processing-made-simple/)
11. [kafka学习笔记：知识点整理](http://www.cnblogs.com/cyfonly/p/5954614.html)