---
title: JavaEE-使用JavaEE开发Web项目
date: 2017-10-2 15:52:10
---

## java web 发展

1999年，推出了 J2EE 1.2 版本，到 2005 年名称变为 Java EE 5，Java EE is currently maintained by Oracle under the Java Community Process。到 2017-9-12 Oracle 宣布将 JavaEE 提交给 Eclipse Foundation。Eclipse 将 Java EE 改名为 EE4J（Eclipse Enterprise for Java），并成为其一个顶级项目。

Java Web 是什么呢？其实就是由 JCP 组织制定的各种规范（JSR），Java EE包含哪些规范呢？参考 [Java™ EE 8 Technologies](http://www.oracle.com/technetwork/java/javaee/tech/index.html) 

所以 JavaEE 平台，其实就是大量的接口，抽象类，注解等。其最终形成一个 jar 包 [javaee-api.jar](http://central.maven.org/maven2/javax/javaee-api/8.0/javaee-api-8.0.jar), 目前 JavaEE 8, 这个包的大小约为 1.9M , 里面将 JavaEE 所有的规范集合到一起。

自然开发JavaEE web 应用，就需要一个实现了 javaee-api 中的所有功能，glassfish 就是一个参考实现，其实现了 JavaEE 所有的功能。

常见的 JavaEE 容器，JBoss, WebLogic, GlassFish, TomEE

在 JavaEE 中大约有 30 多个规范。tomcat 实现了 Servlet，JSP，EL, JSTL, WebSocket 这5个规范。所以 tomcat 只能称为 Servlet 容器，而不能称为 JavaEE 容器。

同样地，Jetty 也是实现了JavaEE部分的规范。

## Spring 和 JavaEE

Spring 出现自 2003 年，此时 J2EE 1.4 已经出现，但是使用 J2EE 进行编程比较复杂，所以 Spring 的出现是为了简化 Java Web 编程，但是 Spring 的各种组件也实现了部分 JSR 。

> Over time, the role of Java EE in application development has evolved. In the early days of Java EE and Spring, applications were created to be deployed to an application server. Today, with the help of Spring Boot, applications are created in a devops- and cloud-friendly way, with the Servlet container embedded and trivial to change. As of Spring Framework 5, a WebFlux application does not even use the Servlet API directly and can run on servers (such as Netty) that are not Servlet containers.

## 常用的轻量级容器

* [google/guice](https://github.com/google/guice)
* [picocontainer/picocontainer](https://github.com/picocontainer/picocontainer)
* [spring-projects/spring-framework](https://github.com/spring-projects/spring-framework)

三个容器的比较 [Comparison between Guice, PicoContainer and Spring](http://www.christianschenk.org/blog/comparison-between-guice-picocontainer-and-spring/) , [Benchmark Analysis: Guice vs Spring](https://www.javalobby.org//articles/guice-vs-spring/)

## TomEE

``` bash
## 启动，停止 glassfis
bin/asadmin.bat start-domain
bin/asadmin.bat stop-domain

## 启动，停止 数据库
bin/asadmin.bat start-database
bin/asadmin.bat stop-database

## 查看 asadmin 支持的命令
bin/asadmin.bat list-commands
## 查看 commands 的帮助信息
bin/asadmin.bat <commands> --help

## 启动 glassfish 后就可以使用下面的命令
## 编译 /opt/glassfish4/docs/javaee-tutorial/examples
mvn -Dglassfish.home='C:\cygwin\opt\glassfish4' -Dglassfish.executables.suffix='.bat'
-Dmaven.test.skip=true package
```

现代的 JAVAEE 应用服务器：WildFly, Payara, WebSphere Liberty, Profile and TomEE

## domain 和 多实例

tomcat 可以使用 CATALINA_BASE 来实现多实例，而在 GlassFish 这类 JAVA EE 服务器中通常都有一个 domain 的概念，但是这个 domain 比多实例要复杂的多，但是有一点类似的是部署在同一个 domain 和 同一个 tomcat 实例中的不用应用，通常是共享同样的资源配置。

## 参考
1. [Apache TomEE](http://tomee.apache.org/apache-tomee.html)
2. [Spring to Java EE Migration](http://www.oracle.com/technetwork/articles/java/springtojavaee-522240.html)
3. [Java(TM) EE 8 Specification APIs](https://javaee.github.io/javaee-spec/javadocs/)
4. [Java EE Specifications](https://javaee.github.io/javaee-spec/Specifications)
5. [Java EE 8 First Cup](https://javaee.github.io/firstcup/toc.html)
6. [Java EE 8 Tutorial](https://javaee.github.io/tutorial/toc.html)
7. [Java Platform, Enterprise Edition (Java EE) 7](http://docs.oracle.com/javaee/7/index.html)
8. [Oracle WebLogic Server 12.2.1.3.0](http://docs.oracle.com/middleware/12213/wls/docs.htm)
9. [Java™ EE 8 Technologies](http://www.oracle.com/technetwork/java/javaee/tech/index.html)
10. [Java EE Specifications](https://javaee.github.io/javaee-spec/Specifications)
11. [Spring Framework 实现的规范](https://docs.spring.io/spring/docs/5.0.0.RELEASE/spring-framework-reference/overview.html#history-of-spring-and-the-spring-framework)
12. [Java Platform, Enterprise Edition wiki](https://en.wikipedia.org/wiki/Java_Platform,_Enterprise_Edition)
13. [The Eclipse Enterprise for Java Project](https://projects.eclipse.org/projects/ee4j/charter)
14. [Jetty and Java EE Web Profile](http://www.eclipse.org/jetty/documentation/current/jetty-javaee.html)
15. [Inversion of Control Containers and the Dependency Injection pattern](https://martinfowler.com/articles/injection.html)
16. [Why Use the Spring Framework?](http://www.wrox.com/WileyCDA/Section/Why-Use-the-Spring-Framework-.id-130098.html)
17. [How to use Tomcat 8.5.x and TomEE 7.x with Eclipse?
](https://stackoverflow.com/questions/37024876/how-to-use-tomcat-8-5-x-and-tomee-7-x-with-eclipse)
18. [Apache Tomcat Versions](http://tomcat.apache.org/whichversion.html)
19. [Java EE SDK Downloads](http://www.oracle.com/technetwork/java/javaee/downloads/java-archive-downloads-eesdk-419427.html)
20. [Java EE—the Most Lightweight Enterprise Framework?](https://community.oracle.com/docs/DOC-1008823)
21. [Stop saying “heavyweight”](https://blog.sebastian-daschner.com/entries/stop_saying_heavyweight)
22. [Unit Testing for Java EE](http://www.oracle.com/technetwork/articles/java/unittesting-455385.html)
23. [Tomcat 多实例](http://tomcat.apache.org/tomcat-9.0-doc/introduction.html#Directories_and_Files)