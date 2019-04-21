---
title: zheng-在eclipse中安装运行
date: 2017-7-1 20:55:46
---

## 安装外部依赖

### mysql

### zookeeper
需要改 conf 目录下配置文件的名称

### redis

redis 需要设置密码

## 下载源码

使用 git clone 到本地，然后使用 eclipse 的 导入 Maven 项目，将项目导入。可能会有报错，应该是 xml, jsp 等文件检验的问题，可以将检验关闭。

## 编译安装到本地仓库

``` cmd
mvn -Dmaven.test.skip=true install
```

## 运行 zheng-upms-server

### 启动依赖

1. mysql

	创建一个名称为 zheng 的数据库，导入 zheng.sql 到 zheng 数据库。
	
2. zookeeper
3. redis

### 修改配置

profiles/dev.properties

使用  com.zheng.common.util.AESUtil.AESEncode 方法对密码进行加密。

添加到配置文件中。

mysql 和 redis 的密码。

### 启动 ZhengUpmsRpcServiceApplication

执行 com.zheng.upms.rpc.ZhengUpmsRpcServiceApplication.main

### 启动 Server

在 zheng-upms-server 的 pom.xml 上执行 jetty:run 命令。