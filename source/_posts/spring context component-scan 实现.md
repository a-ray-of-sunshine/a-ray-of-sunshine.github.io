---
title: spring context component-scan 实现
date: 2016-6-16 15:51:10
tags: [spring, context, component-scan]
---

## 内部解析类

spring 使用以下标签可以实现组件（bean）的自动扫描和注册


	<context:component-scan base-package="org.package.samples" />

包 org.package.samples 中使用 @Component 等注解的类将会被自动注册成 bean，这个过程是如何完成的。

### component-scan 的命名空间

它的命名空间是 

	xmlns:context="http://www.springframework.org/schema/context"

对应的命名空间的注册处理器是

	org.springframework.context.config.ContextNamespaceHandler

查看其中的 init 方法

``` java
registerBeanDefinitionParser("component-scan", new ComponentScanBeanDefinitionParser());
```

由此可知，component-scan 标签的解析由类**org.springframework.context.annotation.ComponentScanBeanDefinitionParser**完成。

查看这个类的源代码查看。调用了

	org.springframework.context.annotation.ClassPathBeanDefinitionScanner.doScan

方法来完成包的扫描。这个方法的核心代码如下：

``` java
Set beanDefinitions = new LinkedHashSet();
	// 扫描配置的所有的包 
for (String basePackage : basePackages) {
	// 扫描出符合条件的 bean 
	Set candidates = findCandidateComponents(basePackage);
	// 注册这些 candidates 成 bean
	for (BeanDefinition candidate : candidates) {
		ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(candidate);
		candidate.setScope(scopeMetadata.getScopeName());
		String beanName = this.beanNameGenerator.generateBeanName(candidate, this.registry);
		// 注册 bean
		if (candidate instanceof AbstractBeanDefinition) {
			postProcessBeanDefinition((AbstractBeanDefinition) candidate, beanName);
		}
		if (candidate instanceof AnnotatedBeanDefinition) {
			AnnotationConfigUtils.processCommonDefinitionAnnotations((AnnotatedBeanDefinition) candidate);
		}
		if (checkCandidate(beanName, candidate)) {
			BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(candidate, beanName);
			definitionHolder = AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder,
					this.registry);
			beanDefinitions.add(definitionHolder);
			registerBeanDefinition(definitionHolder, this.registry);
		}
	}
}
return beanDefinitions;
```

可以看到上面的部分
``` java
Set candidates = findCandidateComponents(basePackage);
```
这个调用应该就是查看 bean 的方法。

这个方法的核心代码：

``` java
Set candidates = new LinkedHashSet();
try {
	// 这个 package 是
	// packageSearchPath:
	// classpath*:packageSearchPath/**/*.class
	// 然后，使用 Resources来加载这些资源。
	String packageSearchPath = "classpath*:" + resolveBasePackage(basePackage) + "/" + this.resourcePattern;
	Resource[] resources = this.resourcePatternResolver.getResources(packageSearchPath);
	boolean traceEnabled = this.logger.isTraceEnabled();
	boolean debugEnabled = this.logger.isDebugEnabled();
	// 扫描 resource，其实就是 package 中的 class
	for (Resource resource : resources) {
		if (traceEnabled) {
			this.logger.trace("Scanning " + resource);
		}
		if (resource.isReadable()) {
			try {
				MetadataReader metadataReader = this.metadataReaderFactory.getMetadataReader(resource);
				if (isCandidateComponent(metadataReader)) {
					ScannedGenericBeanDefinition sbd = new ScannedGenericBeanDefinition(metadataReader);
					sbd.setResource(resource);
					sbd.setSource(resource);
					if (isCandidateComponent(sbd)) {
						if (debugEnabled) {
							this.logger.debug("Identified candidate component class: " + resource);
						}
						candidates.add(sbd);
```

这个方法将 生成 ScannedGenericBeanDefinition。最终注册会被注册到容器中。最终 这些 bean 将会被注册到 context 中。









