---
title: kafka启动过程
date: 2016-6-12 16:11:10
tags: [kafka]
---

## kafka.Kafka

通过 kafka 的启动脚本可知，kafka server 的启动由 kafka.Kafka 类的完成。核心代码如下：

``` java
  val props = Utils.loadProps(args(0))
  
  // 载入，解析 配置文件 properties
  val serverConfig = new KafkaConfig(props)
  KafkaMetricsReporter.startReporters(serverConfig.props)
  val kafkaServerStartable = new KafkaServerStartable(serverConfig)

  // attach shutdown handler to catch control-c
  Runtime.getRuntime().addShutdownHook(new Thread() {
    override def run() = {
      kafkaServerStartable.shutdown
    }
  })

  kafkaServerStartable.startup
  kafkaServerStartable.awaitShutdown
```