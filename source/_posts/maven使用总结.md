---
title: maven使用总结
date: 2017-8-9 11:08:11
---


## 配置调试源码

eclipse 中所有 Launch 的配置都在这里。

workspace\.metadata\.plugins\org.eclipse.debug.core\.launches

指定源码的配置

``` xml
<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<launchConfiguration type="org.eclipse.jdt.launching.localJavaApplication">
<listAttribute key="org.eclipse.debug.core.MAPPED_RESOURCE_PATHS">
<listEntry value="/nebula-web/src/main/java/com/cxy/Main.java"/>
</listAttribute>
<listAttribute key="org.eclipse.debug.core.MAPPED_RESOURCE_TYPES">
<listEntry value="1"/>
</listAttribute>
<stringAttribute key="org.eclipse.jdt.launching.CLASSPATH_PROVIDER" value="org.eclipse.m2e.launchconfig.classpathProvider"/>
<stringAttribute key="org.eclipse.jdt.launching.MAIN_TYPE" value="com.cxy.Main"/>
<stringAttribute key="org.eclipse.jdt.launching.PROJECT_ATTR" value="nebula-web"/>
<stringAttribute key="org.eclipse.jdt.launching.SOURCE_PATH_PROVIDER" value="org.eclipse.m2e.launchconfig.sourcepathProvider"/>
</launchConfiguration>

```


maven 的运行时日志。

workspace\.metadata\.plugins\org.eclipse.m2e.logback.configuration

使用 maven 管理的项目，一定需要在 window -- Preferences -- Maven 界面上勾选 Download Source and JavaDoc. 然后 update Project. 等源码下载完毕，构建完成之后。

再创建一个运行任务。此时就会自动将 Maven 管理的依赖全部添加到 Source Path中。调试的时候就可以看到源码了。

**在 eclipse 中每运行一个程序其实，就是创建了一个对应的 <app>.launch 配置，这个配置的生成就在运行的那一刻，所以可能会出现其它的原因导致程序无法和源码关联，这时对整个项目 clean 一下，重新编译 update, 完成之后，将之前生成的 launch 删除，重新创建一次应该就正常了。**


## 解决 maven 生成的 web 项目导入之后 web.xml 中 web-app 无法改变 version

在 maven 中设置编译器级别

``` xml
<plugin>
	<groupId>org.apache.maven.plugins</groupId>
	<artifactId>maven-compiler-plugin</artifactId>
	<version>3.1</version>
	<configuration>
		<source>${maven.compiler.source}</source>
		<target>${maven.compiler.target}</target>
		<encoding>${project.encoding}</encoding>
	</configuration>
</plugin>
```

将 web.xml 改成 

``` xml
<web-app xmlns="http://java.sun.com/xml/ns/javaee"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://java.sun.com/xml/ns/javaee
                      http://java.sun.com/xml/ns/javaee/web-app_3_0.xsd"
  version="3.0">
```

此时将项目从 eclipse 中移除，然后重新添加，就不会报错了。

## 使用 Nexus 搭建 Maven 私服

**如果安装 Nexus 3.x 则需要 JDK 1.8**
**外部访问需要关闭防火墙**

``` bash
useradd nexus

rpm -ivh jdk-8u144-linux-x64.rpm
tar axvf nexus-3.6.0-02-unix.tar.gz
mv nexus-3.6.0-02 nexus
chown -R nexus:nexus nexus

vi /opt/nexus/bin/nexus.rc
## run_as_user="nexus"
vi /opt/nexus/bin/nexus.vmoptions
## -XX:LogFile=/home/nexus/sonatype-work/nexus3/log/jvm.log
## -Dkaraf.data=/home/nexus/sonatype-work/nexus3
## -Djava.io.tmpdir=/home/nexus/sonatype-work/nexus3/tmp
## 配置为服务
sudo ln -s $NEXUS_HOME/bin/nexus /etc/init.d/nexus
cd /etc/init.d
sudo chkconfig --add nexus
sudo chkconfig --levels 345 nexus on
sudo service nexus start
```

配置 maven 在 settings.xml 中添加如下配置

``` xml
<settings>
  <mirrors>
    <mirror>
      <!--This sends everything else to /public -->
      <id>nexus</id>
      <mirrorOf>*</mirrorOf>
      <url>http://localhost:8081/repository/maven-public/</url>
    </mirror>
  </mirrors>
  <profiles>
    <profile>
      <id>nexus</id>
      <!--Enable snapshots for the built in central repo to direct -->
      <!--all requests to nexus via the mirror -->
      <repositories>
        <repository>
          <id>central</id>
          <url>http://central</url>
          <releases><enabled>true</enabled></releases>
          <snapshots><enabled>true</enabled></snapshots>
        </repository>
      </repositories>
     <pluginRepositories>
        <pluginRepository>
          <id>central</id>
          <url>http://central</url>
          <releases><enabled>true</enabled></releases>
          <snapshots><enabled>true</enabled></snapshots>
        </pluginRepository>
      </pluginRepositories>
    </profile>
  </profiles>
  <activeProfiles>
    <!--make the profile active all the time -->
    <activeProfile>nexus</activeProfile>
  </activeProfiles>
</settings>
```

## 修改 Maven 默认绑定的插件配置

例如，现在有一个需求就是要将所有在 src/main/resources 目录下的文件拷贝到 ${project.build.outputDirectory}/conf 目录下面。

Maven 默认将 maven-resources-plugin 这个插件的 resources goal 绑定到 Maven 生命周期中的 process-resources 这个 pharse. 其默认的id 是 defalut-resources

``` xml
<plugin>
<artifactId>maven-resources-plugin</artifactId>
<executions>
<execution>
    <id>default-resources</id>
	<configuration>
    	<outputDirectory>${project.build.outputDirectory}/conf</outputDirectory>
	</configuration>
</execution>
</plugin>
```

上面的配置表示将默认的 resouces 目录修改到 ${project.build.outputDirectory}/conf 下面。

同理，我们就可以使用 `default-<goalName>` 这种方法配置 execution, 这样就可以修改Maven默认插件的配置。

> each mojo bound to the build lifecycle via the default lifecycle mapping for the specified POM packaging will have an execution Id of default-<goalName> assigned to it, to allow configuration of each default mojo execution independently.

## $. 参考
1. [Introduction to the Dependency Mechanism](https://maven.apache.org/guides/introduction/introduction-to-dependency-mechanism.html)
2. [mvn repository](https://www.mvnrepository.com/)
3. [org.eclipse.m2e.launching](https://github.com/eclipse/m2e-core/tree/master/org.eclipse.m2e.launching)
4. [org.eclipse.m2e.jdt](https://github.com/eclipse/m2e-core/tree/master/org.eclipse.m2e.jdt)
5. [Nexus Installation doc](https://help.sonatype.com/display/NXRM3/Installation)
6. [Download Nexus](https://www.sonatype.com/download-oss-sonatype)
7. [jdk 1.8](http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html)
8. [Configuring Apache Maven Using Nexus](https://help.sonatype.com/display/NXRM3/Maven+Repositories#MavenRepositories-ConfiguringApacheMaven)
9. [Guide to Configuring Default Mojo Executions](https://maven.apache.org/guides/mini/guide-default-execution-ids.html)