---
title: spring mvc 的初始化
date: 2016-6-16 17:31:06
tags: [spring, mvc]
---

## spring 及 springMVC 在 web项目中的配置

``` xml
	<!-- The definition of the Root Spring Container shared by all Servlets and Filters -->
	<context-param>
		<param-name>contextConfigLocation</param-name>
		<param-value>/WEB-INF/spring/root-context.xml</param-value>
	</context-param>
	
	<!-- Creates the Spring Container shared by all Servlets and Filters -->
	<listener>
		<listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
	</listener>
    
	<!-- Processes application requests -->
	<servlet>
		<servlet-name>appServlet</servlet-name>
		<servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
		<init-param>
			<param-name>contextConfigLocation</param-name>
			<param-value>/WEB-INF/spring/appServlet/servlet-context.xml</param-value>
		</init-param>
		<load-on-startup>1</load-on-startup>
		<async-supported>true</async-supported>
	</servlet>
```

由配置可知，主要是下面两个类来完成，spring 容器的初始化：

* org.springframework.web.context.ContextLoaderListener

* org.springframework.web.servlet.DispatcherServlet

### ContextLoaderListener 初始化

整个初始，由下面的方法完成：
org.springframework.web.context.ContextLoader.initWebApplicationContext(ServletContext servletContext) 

其核心代码如下：

``` java
if (servletContext.getAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE) != null) {
	throw new IllegalStateException(
			"Cannot initialize context because there is already a root application context present - check whether you have multiple ContextLoader* definitions in your web.xml!");
}

if (this.context == null) {
	this.context = createWebApplicationContext(servletContext);
}

if (this.context instanceof ConfigurableWebApplicationContext) {
	ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) this.context;
	if (!(cwac.isActive())) {
		if (cwac.getParent() == null) {
			ApplicationContext parent = loadParentContext(servletContext);
			cwac.setParent(parent);
		}
		configureAndRefreshWebApplicationContext(cwac, servletContext);
	}
}
servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, this.context);

```

初始化步骤：

1.  判断 servletContext 中是否存在 WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE 的对象，如果存在，则直接抛异常。

2. 判断 this.context 是否为空，如果为空则 创建。

3. 将创建好的 context set 到 servletContext 中。

### DispatcherServlet 初始化

初始化过程由下面的方法完成：
org.springframework.web.servlet.FrameworkServlet.initServletBean()方法完成。 

核心代码：

``` java
// 这个方法就是从 servletContext 中 获取
// WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE
// 对象，这个对象将被 会被设置成 当前 context 中的 parent
WebApplicationContext rootContext = WebApplicationContextUtils.getWebApplicationContext(getServletContext());
WebApplicationContext wac = null;

if (this.webApplicationContext != null) {
	wac = this.webApplicationContext;
	if (wac instanceof ConfigurableWebApplicationContext) {
		ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) wac;
		if (!(cwac.isActive())) {
			if (cwac.getParent() == null) {
				cwac.setParent(rootContext);
			}
			configureAndRefreshWebApplicationContext(cwac);
		}
	}
}
```