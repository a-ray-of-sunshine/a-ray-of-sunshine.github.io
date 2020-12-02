---
title: 陈兴云-Java工程师-简历
---

### 基本信息
* 陈兴云
* Java开发工程师
* 工作经验:6年
* 本科 陕西理工大学 数学与应用数学
* Email： 884105526@qq.com
* 电话/微信：17611760935
* blog: http://a-ray-of-sunshine.github.io/

### 自我评价

热爱技术，喜欢研究源码，对于新技术的学习有一套基本的方法论，能够快速学习并应用。

工作中常用的框架的源码都喜欢研究，了解其运行和设计机制，掌握了许多设计思路。研究框架源码还有一个好处就是加深对设计模式的理解 ，对编码设计有极大的帮助。

对 JVM 的底层实现比较感兴趣。在 win7 + cygwin 环境下编译成功了 Jamvm 和 GNU Classpath, 并可以正常执行 class 文件。然后使用 visual stdio 搭建了 Jamvm 的源码调试环境。大学的时候学过C语言，Jamvm 的源码清晰简洁，通过研究 Jvm 的运行原理，对 Java 代码的执行环境有了深刻的认识。

能够阅读java 字节码。通过反汇编工具和JVM指令文档，掌握了 class 文件的结构。有助于理解语言结构。基于 JVM 的语言 Clojure，通过反汇编，理解了它的设计和运行机制。

### 专业技能

* 熟练使用Spring MVC、Spring、Mybatis等开源框架,了解研究过其源码。
* 熟练使用Redis/Memcached缓存数据库。
* 熟悉分布式系统开发,熟练掌握使用Dubbo/Thrift分布式服务框架以及Kafka消息队列。
* 熟练使用Java高并发和多线程编程技术。
* 熟练使用Linux系统常用命令,掌握Tomcat, Jetty等Web容器及Nginx原理和常用配置。
* 熟练掌握使用MySQL、Oracle等主流关系数据库,并且能够进行数据库查询语句的简单优化。
* 熟练掌握使用Maven项目管理技术。
* 熟悉svn、git等版本管理工具。
* 熟悉hadoop生态，熟练掌握使用 storm, hadoop, hdfs, hive

### 工作经历

#### 北京趣拿信息技术有限公司（2018年5月 ~ 至今）

* 市场部广告投放效果平台

	整合公司各业务线收益数据，流量数据，投放成本数据。通过大数据平台进行清洗汇总加工处理。进行广告投放效果分析，投放渠道监测，成本核算。主要功能有 渠道成本的管理和维护 渠道ROI报表汇总 收益 成本 流量 同比 环比分析。

	* 负责投放效果分析平台的后端研发
	
		参与项目的整体规划与设计，使用 spring + spring mvc + mybatis 搭建后端框架。完成投放渠道管理，ROI报表下载，邮件发送功能。协助组内成员解决开发过程中遇到的问题。
    
    * 维护和优化现有功能模块
    
		渠道基础元数据数据量巨大，多在千万级别。查询效率底下，对数据实体，表结构进行了重新设计，优化了存储结构，减少了数据冗余。在web层添加了缓存的，提高了查询效率。

#### 北京海天起点技术服务股份有限公司（2016年2月 ~ 2018年5月）

* 数据库中间件监控项目 (2016年10月 ~  2018年5月)

	项目主要功能是对 Oralce 数据库和 Weblogic 中间件进行监控。采集监控目标的运行时状态，对数据进行加工，过滤处理。实现监项告警，运行状态展示。

    * 负责设计和实现采集模块架构和实现

		采集模块是整个产品的数据来源。其功能包括：数据采集，加工处理，实时告警。将整个模块划分为四个组件：a. 采集，b. 加工过滤 c.告警，d.入库。
    
    	划分为多个组件，降低了功能之间的耦合。提高了代码的复用性。同时模块功能更加专一。对每个组件按业务特点进行了优化。采集组件面对的目标类型有 Oracle 和 Weblogic 两种，同时监控的数量也比较大，采集频率比较高。为了提高效率，使用线程池加快了效率。加工过滤和告警模块，需要外部的配置信息，而这些配置信息，基本是静态的，所以直接加载到缓存中。减少了数据库压力，提高了查询速度。
        
        容错机制，采集模块频率比较高，面对目标和网络环境也比较复杂，可能出现采集某个目标的时候，得不到响应，从而响应整个后续的采集。所以设计了超时容错机制。对查询指定超时时间。如果没有响应，则直接加入到超时队列中。然后有一个线程池专门来处理这些超时任务。

    * 运行时状态监控吞吐量

		使用 Java Metrics 进行数据埋点。其自身提供多种展示方法。可以基于这些数据对采集模块的配置进行优化。例如：线程池吞吐量，线程超时时间等。
    
* 应用性能管理项目 (2016年2月 ~ 2016年10月)

	应用性能管理项目的主要功能是采集web应用的访问流量以及SQL查询等信息，分析出应用访问的来源，访问的操作系统，浏览器，等等的不同维度的数据，然后对这些数据进行性能，活跃度，错误率三个指标进行分析。最终得出关于吞吐量，响应延时等耗费较大的请求URL，从而得出对监控系统进行访问性能方面的优化建议。
    
     * 负责数据库设计

		按照数据特性对数据进行的动态，静态，系统数据进行了分类。降低了数据的冗余，增强了数据的逻辑关联性。
                    
		系统最终要对类数据进行分析处理。所以清晰的数据关系，对于后续的数据处理分析非常重要。
                    
		系统产生数据的速率大约在 1G/1hour  对数据进行分类处理。非常明显地降低的存储空间。系统数据就是将采集数据中，可以确定不变的数据，预先内置成 sys_* 类的表。例如 操作系统，浏览器等信息。静态表 static_* 类的表，这类表的数据其实是动态产生的，但是随着系统运行一段时间之后，这类数据就逐渐稳定不变了。例如访问系统的客户端IP，MAC地址等。
    
     * 负责实时流消息处理模块的实现

		其功能是对从端口镜像过来的数据进行处理。涉及到数据的分类，滤过，清洗。数据源来自 Kafka. 为了提高整个模块的处理效率，在设计 Spout 的时候，基于 Kafka consumer 分组进行消费。提高了消费速率。
                
		在 storm 中进行分析处理。storm 提供了一个实时处理数据的平台，但在实际的业务场景中，通常是需要定时对数据进行入库操作，所以需要有一种机制用来实现，批量提交。通过查找阅读一些博客资料（Implementing Real-Time Trending Topics with a Distributed Rolling Count Algorithm in Storm），实现了基于滑动窗口的统计模式。

#### 西安锐图软件开发有限公司（2013年7月 ~ 2016年2月）

* 教师综合数据管理与服务平台 (2014年11月 ~ 2016年2月)

	这个项目是高校教师相关的数据（教学，科研，人事等）同步，采集，加工，管理和服务系统。面向的用户：高校教师，高校的业务行政人员，以及校领导。项目采用 Spring + Spring MVC + Mybatis 实现。
	
	* 数据导入模块

	导入的数据分为多种类型，而且将来可以会有其它格式的数据源出现，按照 OOP 的思想设计了一个可扩展的数据导入模块。如果有新的数据源，只需要继承相应的抽象类，实现抽象方法。

   	* 动态创建数据库表

	功能类似于 Navicat 的创建表，修改表功能。
	
* 客户积分管理系统 (2013年7月 ~ 2014年10月)

	按多个指标对客户进行分级，按照业务规则，将客户进行分级，从而给出相应的积分，系统负责进行积分的规则的制定，积分的换算，积分的兑换等。
    
    我在这个项目中负责，积分规则的制定模块的开发，该模块的主要功能是按照业务需求，采集相关的规则计算数据，例如年限，比率等，然后生成相应的积分规则。