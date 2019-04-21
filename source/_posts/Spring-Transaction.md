---
title: Spring-Transaction
date: 2017-6-28 18:41:35
---

## 事务传播

事务的传播特性。当 spring 托管事务的时候。在 Service 层的实现中，可能会出现，一个 Service 方法的实现，可能依赖于其它的 Service的多个方法。而这几个方法都是存在事务的，那么那些被嵌套调用的方法上的事务，如何创建呢，是挂起当前的事务，创建一个新的事务，还是直接就在当前事务中执行。这个就是事务的传播。


## 事务托管之后

一旦事务被托管，则 dao 层中就不应该再创建事务。

如果使用的Session 是当前事务中的Session，也就是，通过 getCurrentSession 获得到的Session，则不应该使用这个 Session 对象来创建事务对象，并开启一个事务，因为事务对象已经被开启了，所以调用 beginTransaction 并没什么用，但是此时事务是否需要被关闭呢，不能关闭，一旦关闭，之后的事务就不起使用了。

如果是使用 SessionFactory.openSession() 获得的事务，则与当前托管的事务没有关系，则可以正常使用这个Session对象的事务。

## Hibernate 中的 Session

current Session 的绑定。

org.hibernate.context.ThreadLocalSessionContext

## Spring MVC 参数映射

### 自定义参数

在 Controller 层，当参数为对象的时候，Spring MVC 会自动将 req 获取到的参数进行装配。但是有的时候，我们需要自定义这个装配过程。例如在前后端分离的情况下，前台传递过来的参数名称可能和后台的 POJO 的字段名称无法对应。此时就需要自定义这个装配过程。

其方法可以参考： [自定义Spring MVC3的参数映射和返回值映射 + fastjson](http://www.cnblogs.com/daxin/p/3296493.html)

### 自定义返回值

同理，前后端分离的时候，有可以 POJO 对象，已经在代码大量使用了。

而在和前台对接的时候，POJO 的字段名称无法对应，则可以使用下面的注解的来解决。

``` java
@JsonProperty("name")
private String source;

@JsonProperty("value")
private long reqCount;
```

## 参考
1. [Spring Transaction](https://docs.spring.io/spring/docs/current/spring-framework-reference/html/transaction.html)
2. [Hibernate ORM 5.2.10.Final User Guide](http://docs.jboss.org/hibernate/orm/5.2/userguide/html_single/Hibernate_User_Guide.html)
3. [自定义Spring MVC3的参数映射和返回值映射 + fastjson](http://www.cnblogs.com/daxin/p/3296493.html)