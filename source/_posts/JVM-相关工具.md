---
title: JVM-相关工具
date: 2017-3-30 11:37:31
---

## 配置 jmx

在 jvm 启动的时候添加以下参数

``` bash
JAVA_OPTS="-Dcom.sun.management.jmxremote=true
           -Dcom.sun.management.jmxremote.port=12345
           -Dcom.sun.management.jmxremote.authenticate=false
           -Dcom.sun.management.jmxremote.ssl=false"
```

使用 jmx 连接远程机器时，需要关闭防火墙

``` bash
1） 临时生效，重启后复原
开启： service iptables start
关闭： service iptables stop

2） 永久性生效，重启后不会复原
开启： chkconfig iptables on
关闭： chkconfig iptables off
```

## 配置远程调试
``` bash
-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=60001
```

## 查看对象占用情况

``` bash
## 查看内存使用情况
jmap -heap <pid>
## 查看对象占用情况
jmap -histo <pid> 

## 当前需要导出 Jvm 的内存比较大的时候（> 1G）
## 可能会报错， java.io.IOException: Premature EOF
jmap -dump:format=b,file=<heap_dump_filename> <PID>
## 报错使用下面的命令导出 dump 文件
jmap -J-d64 -dump:format=b,file=<heap_dump_filename> <PID>
```

## jvm 参数

堆的总大小和可用堆的比例

### MinHeapFreeRatio

### MaxHeapFreeRatio



## jdk 工具

[JDK Tools and Utilities](http://docs.oracle.com/javase/7/docs/technotes/tools/)

## $. 参考
1. [Monitoring and Management Using JMX Technology](http://docs.oracle.com/javase/7/docs/technotes/guides/management/agent.html)
2. [Java SE Monitoring and Management Guide](http://docs.oracle.com/javase/7/docs/technotes/guides/management/toc.html)
3. [Java Platform Standard Edition 7 Documentation](http://docs.oracle.com/javase/7/docs/)
4. [Monitoring and Management for the Java Platform](http://docs.oracle.com/javase/7/docs/technotes/guides/management/index.html)
5. [Java HotSpot VM Options](http://www.oracle.com/technetwork/java/javase/tech/vmoptions-jsp-140102.html)
6. [java常用选项](http://docs.oracle.com/javase/7/docs/technotes/tools/solaris/java.html)
7. [Java™ Virtual Machine Technology](http://docs.oracle.com/javase/7/docs/technotes/guides/vm/)
8. [Non-standard Java HotSpot VM Options](http://www.oracle.com/technetwork/java/javase/tech/vmoptions-jsp-140102.html)
9. [Java SE 6 HotSpot™ Virtual Machine Garbage Collection Tuning](http://www.oracle.com/technetwork/java/javase/gc-tuning-6-140523.html)
10. [Troubleshooting Guide for Java SE 6 with HotSpot VM](http://www.oracle.com/technetwork/java/javase/toc-135973.html)
11. [Troubleshooting Guide for HotSpot VM](http://docs.oracle.com/javase/7/docs/webnotes/tsg/TSG-VM/html/tools.html)
12. [Java heap dump error with jmap command : Premature EOF](http://stackoverflow.com/questions/36604102/java-heap-dump-error-with-jmap-command-premature-eof)