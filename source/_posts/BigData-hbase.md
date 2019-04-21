---
title: BigData-hbase
date: 2016-12-5 14:48:35
---

## 0. 安装

参考下面的的3文档，进行安装。安装完成之后，启动 hbase

`bin/start-hbase.sh` 命令用来启动 hbase, 启动之后，

使用 jps 命令，可以看到有一个名称为 HMaster 的 JVM 进程，就是 hbase 的进程。

设置 zookeeper 端口

可以在 conf/hbase-site.xml 文件中增加下面的配置。

```
<property>
    <name>hbase.zookeeper.property.clientPort</name>
    <value>2181</value>
</property>
```

## 2. 配置

* hbase.master.info.port

	> The port for the HBase Master web UI. Set to -1 if you do not want a UI instance run.
	>
	> Default
	>
	> 16010

* hbase.regionserver.info.port

	> The port for the HBase RegionServer web UI Set to -1 if you do not want the RegionServer UI to run.
	> 
	> Default
	> 
	> 16030
 	
访问 16010 端口，可以查看 hbase 的 WEB UI 界面。
 	
## $. 参考
1. [hbase 文档](http://hbase.apache.org/book.html)
2. [hbase 下载](http://mirrors.cnnic.cn/apache/hbase)
3. [hbase 安装文档](http://hbase.apache.org/book.html#quickstart)
4. [hbase使用独立的zookeeper](http://www.aboutyun.com/thread-7451-1-1.html)
5. [hbase javadoc](http://hbase.apache.org/apidocs/index.html)
6. [HBase数据存储格式](http://blog.csdn.net/scgaliguodong123_/article/details/46745825)
7. [HBase的存储格式是什么](http://blog.itpub.net/15498/viewspace-1876022/)
8. [HADOOP HBASE适合存储哪类数据](http://www.cnblogs.com/chenjingjing/archive/2010/01/26/1656869.html)
9. [Hbase原理、基本概念、基本架构](http://blog.csdn.net/woshiwanxin102213/article/details/17584043)