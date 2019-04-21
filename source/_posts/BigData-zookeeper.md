---
title: zookeeper
date: 2016-06-11 10:16:07
tags: [zookeeper, apache]
---

## 1. zookeeper简介

zookeeper 进行集群的目地是提供高可用的服务。

znode: org.apache.zookeeper.server.DataNode

整个目录层级结构： org.apache.zookeeper.server.DataTree


## 2. 安装

1. 下载安装包

	最新稳定版本： [zookeeper](https://mirrors.tuna.tsinghua.edu.cn/apache/zookeeper/stable/)
	
2. 安装文档

	[zookeeperStarted](http://zookeeper.apache.org/doc/r3.4.9/zookeeperStarted.html)
	
3. 系统需求

	[System Requirements](http://zookeeper.apache.org/doc/r3.4.9/zookeeperAdmin.html#sc_systemReq)

### 安装

```
tickTime=2000
dataDir=/var/zookeeper
clientPort=2181
initLimit=5
syncLimit=2
server.1=zoo1:2888:3888
server.2=zoo2:2888:3888
server.3=zoo3:2888:3888
```

在多台机器上安装 zookeeper 集群时，配置好相关的配置之后，需要在 dataDir 目录下面创建一个 myid 文件，这个文件中包含当前机器的编号。

> The myid file consists of a single line containing only the text of that machine's id. So myid of server 1 would contain the text "1" and nothing else. The id must be unique within the ensemble and should have a value between 1 and 255.

安装集群的时候，配置文件完全相同，所以当在 zoo1 机器上启动 zookeeper 的时候，zookeeper 并不知道自己是属于哪个 server 的，所以需要在 dataDir 目录下创建一个名为 myid 的文件，其中包含 zoo1 对应的 数字 1 (server.1)。对应的此时启动的 zookeeper 就可以使用其所对应的端口。

## 参考
1. zookeeper: [zookeeper](http://zookeeper.apache.org/)
2. docs: [zookeeper-3.4.8 docs](http://zookeeper.apache.org/doc/r3.4.8/)
3. wiki: [zookeeper wiki](https://cwiki.apache.org/confluence/display/ZOOKEEPER/Index)
4. [zookeeper配置相关](http://zookeeper.apache.org/doc/r3.4.9/zookeeperAdmin.html)