---
title: Qunar基础框架-QSchedule
时间: 2018-8-3 11:33:37
---

## 简介

## QSchedule 使用方式

核心是获取 `SchedulerProvider` 的实例

### 基于 Java 代码

``` java
// 1. 创建并初始化 SchedulerProvider
SchedulerProvider schedulerProvider = new SchedulerProvider();
schedulerProvider.init();

// 2. 实现具体的任务
//    任务必须实现 qunar.tc.schedule.Worker 接口
public class MyJobWorker implements Worker{
   public void doWork(Parameter parameter) {
      //do your job
   }
}

// 3. 注册任务
schedulerProvider.schedule("你的job名字", new MyJobWorker()));

// 4. 在应用退出前清理资源
schedulerProvider.destroy();
```

### 基于配置文件

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:qschedule="http://www.qunar.com/schema/qschedule"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
       http://www.qunar.com/schema/qschedule http://www.qunar.com/schema/qschedule/qschedule.xsd
       http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd">
 
    <!-- 普通的bean，用来执行QSchedule任务 -->
    <bean id="orderTask" class="com.qunar.hotel.tts.OrderTask">
       <property name="orderDAO" ref="orderDAO" />
    </bean>
    
    <qschedule:config port="被调度的端口号(可选，默认是20070)"/>
    <qschedule:task id="任务名称" ref="orderTask" method="processOrders"/>
</beans>
```

### 基于注解

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:qschedule="http://www.qunar.com/schema/qschedule"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
       http://www.qunar.com/schema/qschedule http://www.qunar.com/schema/qschedule/qschedule.xsd
       http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd">

    <!-- 使用annotation方式配置 -->
    <qschedule:config port="20070"/>
</beans>
```

在 application.xml 做了上面的配置后, 就可以直接在方法上使用 `QSchedule` 注解, 方法就可以被注册为
任务

### auto-configuration 

在 QSS 项目中对 QSchedule 进行了封装, 采用了 `@Configuration` 机制，进行了默认配置

可以直接在需要注册的任务方法上使用 `@QSchdule` 注解.

##  内部实现

### 与 Spring 框架的集成

`SchedulerProvider` 类的优化：

* 加入状态概念 NEW INIT START DESTORY
* 将 init 方法上的 `@PostConstruct` 注解去掉
* 将 destory 方法上的 `@PreDestroy` 注解去掉
* `doScheduleResultWorker` 中验证当前状态，如果没有进行初始化，则调用 init 方法

### 任务注册

`JobRegistryService`

需要将

### 任务调度

`qunar.tc.qschedule.executor.ScheduleMessageDistributor.addListener`
