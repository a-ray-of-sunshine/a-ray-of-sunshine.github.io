---
title: nebula-创建项目
date: 2017-7-23 17:13:30
---

## nebula framework

一个基于 spring 的 java web 框架

目标：

* 易于扩展
* 前后端分离
* 实现基本的用户，权限模型
* 集成常用工具，例如： 定时调度，Web Service
* 支持 server 向 client 端的消息推送，例如推送待办事项。
* Spring 支持多种环境的配置 dev, test, prodect
* 自动创建初始化数据库
* **自动生成  dao, service, controller 层的文件，以及基本方法**, 利用 maven 的 generate-source 的生命周期，执行生成源码的任务。
* 引入状态度量框架，方便地查看 JVM 中资源的使用情况，例如：线程池，各种连接池的使用情况。http://metrics.dropwizard.io
* 业务层引入缓存。
* 静态代码检测工具，FindBugs 考虑将其集成到 maven 中。编译完成之后，调用 findbugs, 查找问题。
* 学习 Apache Commons 系统工具类。
* xml 配置解析库，org.apache.commons.configuration2
* 学习 Guava
* 引入工作流


可以参考 https://mvnrepository.com/open-source 引入库。


## 项目创建

### 创建父工程

``` bash
mvn archetype:generate \
-DarchetypeGroupId=org.codehaus.mojo.archetypes \
-DarchetypeArtifactId=pom-root \
-DarchetypeVersion=RELEASE \
-DgroupId=com.nebula \
-DartifactId=nebula-framework \
-Dversion=0.1
```

### 创建子工程

cd 到上面创建的 nebula-framework 目录中，执行：

``` bash
## 创建 web 类的子项目
mvn archetype:generate \
-DarchetypeGroupId=org.apache.maven.archetypes \
-DarchetypeArtifactId=maven-archetype-webapp \
-DarchetypeVersion=RELEASE \
-DgroupId=com.nebula \
-DartifactId=nebula-web \
-Dversion=0.1

## 创建 jar 类的子项目
mvn archetype:generate \
-DarchetypeGroupId=org.apache.maven.archetypes \
-DarchetypeArtifactId=maven-archetype-quickstart \
-DarchetypeVersion=RELEASE \
-DgroupId=com.nebula \
-DartifactId=nebula-core \
-Dversion=0.1
```

新创建的项目将自动把当前目录当作 parent. 并且会直接修改 当前的 parent pom 文件将子工程添加进来。

### 导入到开发环境中

如果不使用 eclipse 自带的 maven 插件，则使用 下面的命令生成工程文件。

``` bash
mvn eclipse:eclipse
```

如果 eclipse 已经安装了 maven 插件，则使用 导入 maven 项目，即可将项目导入到 eclipse 中。

## 参考
1. [c3p0 docs](http://www.mchange.com/projects/c3p0/)
2. [modelmapper](http://modelmapper.org/)
