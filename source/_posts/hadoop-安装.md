---
title: hadoop-安装
date: 2016-8-26 15:04:33
tags: [hadoop]
---

## hadoop

[hadoop](http://hadoop.apache.org/) 官网

安装教程：

[Hadoop: Setting up a Single Node Cluster.](http://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-common/SingleCluster.html)

### 注意事项

* 设置 JAVA_HOME

	编辑这个文件 etc/hadoop/hadoop-env.sh 将其中的 {JAVA_HOME} 替换成本机安装的 JAVA_HOME 的绝对路径。
	
* 将 localhost 设置成可以无口令 ssh 登录

	``` bash
	$ ssh-keygen -t dsa -P '' -f ~/.ssh/id_dsa
  	$ cat ~/.ssh/id_dsa.pub >> ~/.ssh/authorized_keys
  	$ chmod 0600 ~/.ssh/authorized_keys
	```

* 清空 logs 目录

	如果启动 `sbin/start-dfs.sh` hdfs 报错，可以尝试清空 $HADOOP_HOME/logs

## 使用 Shell Commands 管理 hdfs

使用上面的步骤，启动 hdfs，然后在访问  http://namenode-name:50070/ 可以看到 hdfs 的管理界面。

``` bash
bin/hdfs dfs -help
bin/hdfs dfs -help command
```

The command bin/hdfs dfs -help lists the commands supported by Hadoop shell. Furthermore, the command bin/hdfs dfs -help command-name displays more detailed help for a command. These commands support most of the normal files system operations like copying files, changing file permissions, etc. It also supports a few HDFS specific operations like changing replication of files. For more information see [File System Shell Guide](http://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-common/FileSystemShell.html).


## 启动和关闭 hdfs

1. 安装 hadoop

	下载地址： [hadoop download](http://mirrors.cnnic.cn/apache/hadoop/common/)

2. 解压到指定目录

3. 修改 etc/hadoop/hadoop-env.sh

	设置 JAVA_HOME
	
4. 启动 hdfs

	`sbin/start-dfs.sh`
	
5. 访问 web 管理界面

	http://namenode-name:50070/
	
6. 关闭 hdfs

	`sbin/stop-dfs.sh`

## 

[使用Hadoop提供的API操作HDFS](http://acesdream.blog.51cto.com/10029622/1625446)

[如何使用Java API读写HDFS](http://qindongliang.iteye.com/blog/1982039)

## Hive 读取 protobuf 数据

[How to use Elephant Bird with Hive](https://github.com/twitter/elephant-bird/wiki/How-to-use-Elephant-Bird-with-Hive)

[twitter/elephant-bird](https://github.com/twitter/elephant-bird)

[Use elephant-bird with hive to read protobuf data](http://stackoverflow.com/questions/27791961/use-elephant-bird-with-hive-to-read-protobuf-data)

[Apache Hive docs](https://cwiki.apache.org/confluence/display/Hive/Home)

[hive数据类型和文件格式](http://www.niubua.com/2014/11/28/hive%E6%95%B0%E6%8D%AE%E7%B1%BB%E5%9E%8B%E5%92%8C%E6%96%87%E4%BB%B6%E6%A0%BC%E5%BC%8F/)

[Row Format, Storage Format, and SerDe](https://cwiki.apache.org/confluence/display/Hive/LanguageManual+DDL)


## 搭建 hive 环境

[GettingStarted-RunningHive](https://cwiki.apache.org/confluence/display/Hive/GettingStarted#GettingStarted-RunningHive)

* 首先安装 hadoop , 设置 HADOOP_HOME 环境变量。

	``` bash
	export HADOOP_HOME=<hadoop-install-dir>
	```

* 在 hadoop 中初始化 hive 的工作目录

	``` bash
	$ $HADOOP_HOME/bin/hadoop fs -mkdir       /tmp
  	$ $HADOOP_HOME/bin/hadoop fs -mkdir       /user/hive/warehouse
  	$ $HADOOP_HOME/bin/hadoop fs -chmod g+w   /tmp
  	$ $HADOOP_HOME/bin/hadoop fs -chmod g+w   /user/hive/warehouse
	```

* 设置 HIVE_HOME

	``` bash
	$ export HIVE_HOME=<hive-install-dir>
	```
	
* 初始化 metadata

	[Hive Schema Tool](https://cwiki.apache.org/confluence/display/Hive/Hive+Schema+Tool)
	
	hive 自身需要使用数据库来存储其 metadata 所以在使用之前使用 The Hive Schema Tool 来初始化 hive 的 metadata.
	
	``` bash
	$ schematool -dbType derby -initSchema
	```
	
	如果需要使用的数据数据库，例如使用 mysql 来存储 metadata 则需要进行额外的配置。
	
	[Hadoop Hive安装，配置mysql元数据库](http://blog.csdn.net/x_i_y_u_e/article/details/46845609)

## hadoop 海量日志分析解决方案

[Hadoop学习笔记—20.网站日志分析项目案例（一）项目介绍](http://www.cnblogs.com/edisonchou/p/4449082.html)

[hadoop学习--基于Hive的Hadoop日志分析](http://blog.csdn.net/y521263/article/details/23969745)


## hadoop 集群环境的搭建

### 安装centos

使用 Oracle virtual Box 安装时 centos7， 时注意，设置好内存大小，默认是 256 或者是 512 太小了，安装过程中界面会卡顿，所以最好将内存设置成 1024.

注意：在 Oracle virtual Box 将虚拟机的网络连接方法改成，桥接网卡，这样新建的虚拟机就和当前物理机在同一个网络当中了。

复制虚拟机：点击，控制---复制，即可。复制的虚拟机，由于 mac 地址是一样的所以需要将重新，配置虚拟机的mac地址。设置--网络--高级，MAC 地址。

1. 设置虚拟机后台启动

2. 配置 yum 源

参考：[网易CentOS镜像使用帮助](http://mirrors.163.com/.help/centos.html)

2. 安装下载工具 wget

``` bash
yum install wget
```

3. 安装 rz & sz

``` bash
yum install lrzsz
```

4. 安装 jdk


由于下载时，要选择同意协议，在 wget 上添加下面的参数：

``` bash
cd /tmp

wget --no-check-certificate --no-cookies --header "Cookie: oraclelicense=accept-securebackup-cookie" http://download.oracle.com/otn-pub/java/jdk/7u79-b15/jdk-7u79-linux-x64.tar.gz

mkdir /usr/local/java
tar xvf jdk-7u79-linux-x64.tar.gz -C /usr/local/java
rm jdk-7u79-linux-x64.tar.gz
```

[Downloading Java JDK on Linux via wget is shown license page instead](http://stackoverflow.com/questions/10268583/downloading-java-jdk-on-linux-via-wget-is-shown-license-page-instead)

5. 配置环境变量

	在 /etc/profile.d 目录下创建一个 java-en.sh 文件
	添加以下内容
	
	``` bash
	# java web env vars.
	JAVA_HOME=/usr/local/java/jdk1.7.0_79
	CLASS_PATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
	
	export JAVA_HOME
	export CLASS_PATH
	export PATH=$PATH:$JAVA_HOME/bin
	```
	
	然后，执行 `source java-en.sh`
	
	验证：`java -version` & `javac -version` 可以正常打印出信息，即可。

## CDH 

apache hadoop 的一个发行版本。

[documentation](http://www.cloudera.com/documentation.html)

[manager document](http://www.cloudera.com/documentation/manager/5-1-x.html)

[Cloudera-Manager-Installation-Guide](http://www.cloudera.com/documentation/manager/5-1-x/Cloudera-Manager-Installation-Guide/Cloudera-Manager-Installation-Guide.html)

[cdh document](http://www.cloudera.com/documentation/cdh/5-1-x.html)

[CDH5-Installation-Guide](http://www.cloudera.com/documentation/cdh/5-1-x/CDH5-Installation-Guide/CDH5-Installation-Guide.html)

rpm 包下载地址

[cloudera-manager rpms](http://archive.cloudera.com/cloudera-manager/redhat/6/x86_64/cloudera-manager/3/RPMS/)

1. 设置 yum 仓库源

	```
	wget http://archive.cloudera.com/cm5/redhat/6/x86_64/cm/cloudera-manager.repo

	cp cloudera-manager.repo /etc/yum.repos.d
	```

2. 安装 jdk

	```
	 yum install oracle-j2sdk1.7
	```
	
	安装完成之后，设置 JAVA_HOME, CLASS_PATH

3. 安装 cloudera-manager

	```
	yum install cloudera-manager-daemons cloudera-manager-server
	```
	
4. 设置 	cloudera-manager 的数据库

	1. 安装数据库(默认是 PostgreSQL )
	
		```
	 	yum install cloudera-manager-server-db-2
		```
	
	2. 启动数据库

	
		```
		# 禁用 selinux , 将 SELINUX=disabled
		cat /etc/sysconfig/selinux
		# 需要重新启动机器,上面的禁用才会生效
		shutdown -r now
		service cloudera-scm-server-db start
		```
		
	3. 初始化数据库
	
		```
		/usr/share/cmf/schema/scm_prepare_database.sh
		```
	
		
5. 启动 cloudera-manager

	```
	service cloudera-scm-server start
	```

6. 访问 cloudera-manager

	http://Server host:7180
	
	Username: admin Password: admin



[Enable or Disable SELinux](https://www.centos.org/docs/5/html/5.1/Deployment_Guide/sec-sel-enable-disable.html)

[ScmActive failed to validate CM identity 错误处理办法](https://www.cloudera.com/documentation/enterprise/latest/topics/cm_failover_db.html)

[Managing Clusters with Cloudera Manager 使用Cloudera Manager安装管理集群](http://www.cloudera.com/documentation/manager/5-1-x/Cloudera-Manager-Managing-Clusters/Managing-Clusters-with-Cloudera-Manager.html)

[集群安装失败](http://community.cloudera.com/t5/Cloudera-Enterprise-5-Beta/Got-error-when-install-cdh5-via-Manager5-quot-5-68-168-192-in/m-p/4843#U4843)

[离线安装Cloudera Manager 5和CDH5(最新版5.1.3) 完全教程](http://www.cnblogs.com/jasondan/p/4011153.html)

[A Quick Guide to Using the MySQL Yum Repository](https://dev.mysql.com/doc/mysql-yum-repo-quick-guide/en/)

[CDH5 Hadoop集群完全离线安装步骤总结](http://archive-primary.cloudera.com/cm5/cm/5/)

[Cloudera Manager 5和CDH5离线安装](http://www.361way.com/cloudera-manager-cdh-install/5033.html)

[CDH 5 and Cloudera Manager 5 Requirements and Supported Versions](http://www.cloudera.com/documentation/enterprise/release-notes/topics/rn_consolidated_pcm.html)

[](http://archive.cloudera.com/cdh5/parcels/)

在从结点上安装 Java 和 agent

创建用户

创建 run/cloudera-scm-agent 文件夹


## centos6.5 安装 cdh5.5

### 下载软件包

### 配置centos

1. 设置静态ip

2. 设置hostname

	centos6.5 配置 /etc/sysconfig/network 中的 HOSTNAME 即可
	centos7 需要配置 /etc/hostname 文件
	
	配置内外网卡
	
3. 创建本地 yum 源。
createrepo rpm

将所有使用到的 rpm 都放置到 /usr/local/local-repo/ 中。

createrepo -v /usr/local/local-repo/

https://www.rpmfind.net/linux/RPM/opensuse/13.2/x86_64/createrepo-0.10.3-3.1.4.x86_64.html




1. 创建仓库所在文件夹

mkdir /usr/local/local-repo/

2. 将所有rpm 拷贝到 上面的文件夹

3. 创建仓库

createrepo -v /usr/local/local-repo/

如果，后续有新的rpm 添加进来，则使用上面的命令，刷新。

刷新仓库
createrepo -v --update /usr/local/local-repo/

刷新yum缓存
yum clean all

4. 配置 yum 源

在/etc/yum.repo.d/下创建一个repo文件，文件名可以自定义，但一定要以repo结尾

例如 local-repo.repo
添加如下内容

[local-repo]                    #仓库名称可以自定义
name=This is a local repo    #描述信息
baseurl=file:///usr/local/local-repo/ #这里填写仓库的url，注意 有三个正斜线 
enabled=1                    #是否开启仓库，1为开启，0为关闭
gpgcheck=0                #是否检查gpgkey，1为开启，0为关闭


5. 禁用外部源

CentOS-Base.repo 备份
CentOS-Base.repo 禁用所有需要连接的仓库 enable=0， 默认启用的那几个仓库没有 enable 配置，直接加上即可。

6. 初始化 yum 缓存

yum clean all

7. 验证

yum list | grep clouder*

可以看到本地仓库中rpm

cloudera-manager-agent.x86_64
cloudera-manager-daemons.x86_64
cloudera-manager-server.x86_64
cloudera-manager-server-db-2.x86_64

8. 安装 cloudera

./cloudera-manager-installer.bin --skip_repo_package=1


必须先安装 perl 才可以。
(1/6): perl-5.10.1-141.el6_7.1.x86_64.rpm                                |  10 MB     00:05
(2/6): perl-Module-Pluggable-3.90-141.el6_7.1.x86_64.rpm                 |  40 kB     00:00
(3/6): perl-Pod-Escapes-1.04-141.el6_7.1.x86_64.rpm                      |  33 kB     00:00
(4/6): perl-Pod-Simple-3.13-141.el6_7.1.x86_64.rpm                       | 213 kB     00:00
(5/6): perl-libs-5.10.1-141.el6_7.1.x86_64.rpm                           | 579 kB     00:00
(6/6): perl-version-0.77-141.el6_7.1.x86_64.rpm                          |  52 kB     00:00


### 安装 postgresql 数据库

注意安装顺序：

rpm -ivh postgresql94-libs-9.4.1-1PGDG.rhel6.i686.rpm

rpm -ivh postgresql94-9.4.1-1PGDG.rhel6.i686.rpm

rpm -ivh postgresql94-server-9.4.1-1PGDG.rhel6.i686.rpm

rpm -ivh postgresql94-contrib-9.4.1-1PGDG.rhel6.i686.rpm


需要将cloudera所在的主机自身，也添加到 hosts文件中。

需要关闭 iptables
service iptables stop
chkconfig iptables off

将 拷贝到 /opt/cloudera/parcel-repo

设置本地 CDH 安装源：

[设置本地 CDH 安装源](http://www.cloudera.com/documentation/archive/manager/4-x/4-8-5/Cloudera-Manager-Installation-Guide/cmig_create_local_parcel_repo.html)

注意完成上面的，步骤之后，将拷贝后的文件权限更改成 cloudera-scm用户，

chown -R cloudera-scm:cloudera-scm ./*

 ERROR ParcelUpdateService:com.cloudera.parcel.components.ParcelDownloaderImpl: Unable to retrieve remote parcel repository manifest


[cloudera 最新安装文档](http://www.cloudera.com/documentation/enterprise/latest/topics/quickstart.html)


本地化安装

1. yum install yum-plugin-downloadonly

2. yum install --downloadonly --downloaddir=/tmp <package-name>

3. yum localinstall 列出所有的包上面。


需要在cluster上设置，parcel-repo

/usr/share/cmf/packages/ 位置。


创建一个临时远程仓库：

[Creating a Temporary Remote Repository](http://www.cloudera.com/documentation/enterprise/latest/topics/cm_ig_create_local_parcel_repo.html)

python -m SimpleHTTPServer 8900

安装agent的过程中可以使用 tail -f  /var/log/yum.log 来查看


### 各个界面功能，介绍

1. 群集安装

在 /tmp 目录下有一个scm_prepare_node 开头的文件夹。
里面有一个 scm_prepare_node.sh 脚本。这个脚本的就是在群集安装这个界面所执行的。

整个过程如下：

take_lock
detect_root
detect_distro
detect_scm_server
cloud_specific_configure
setup_blacklist
install_repo_files
install_priority_files
refresh_metadata
install_remote_packages
install_unlimited_jce
configure_agent
start_agent

其中：install_repo_files 这个过程会将 一个名称为 cloudera-manager.repo 的文件，安装到 /etc/yum.repos.d 中。创建了一个仓库 cloudera-manager， 其url 就是：选择存储库时，配置的自定义存储库。

这个界面的功能是安装并启动agent.

2. 正在安装选定 Parcel 

总共有4个过程

* 已下载 --- 表示 master 主机的 /opt/cloudera/ 目录中已经有了 CDH的包
* 已分配 --- master 向各个 cluster-host 发送 Parcel 包，到 /opt/cloudera/parcel-cache 目录中
* 已解压 --- cluster-host 将parcel-cache目录中的CDH解压到一个临时目录中。
* 已激活 --- 不知道？？？其作用有可能是向 master 和 自身主机，报告 CDH 已经成功解压了。

这个过程是安装在 /opt/cloudera/ 中的 parcel.

3. 检查主机正确性

可能会遇到下面的问题：

* hadoop-worker1: IOException thrown while collecting data from host: No route to host

是因为agent机器开启了防火墙
service iptables stop 关闭后重新检测。

* Cloudera 建议将 /proc/sys/vm/swappiness 设置为 0。当前设置为 60。使用 sysctl 命令在运行时更改该设置并编辑 /etc/sysctl.conf 以在重启后保存该设置。您可以继续进行安装，但可能会遇到问题，Cloudera Manager 报告您的主机由于交换运行状况不佳。以下主机受到影响：

* 已启用“透明大页面”，它可能会导致重大的性能问题。版本为“/sys/kernel/mm/transparent_hugepage”且发行版为“{1}”的 Kernel 已将 enabled 设置为“{2}”，并将 defrag 设置为“{3}”。请运行“echo never > /sys/kernel/mm/redhat_transparent_hugepage/defrag”以禁用此设置，然后将同一命令添加到一个 init 脚本中，如 /etc/rc.local，这样当系统重启时就会予以设置。或者，升级到 RHEL 6.5 或更新版本，它们不存在此错误。将会影响到以下主机： 

到此为止，整个集群就安装完成了。

接下来是集群配置

### 集群配置
http://192.168.0.12:7180/cmf/clusters/1/express-add-services/index

数据库设置

默认 postgresql: hive xFWqCmc67A

首次运行:

可能会遇到的问题：

Hive Metastore Database Host must be configured when Hive metastore is configured to use a database other than Derby.

安装hive的时候的问题：

[cdh_ig_hive_metastore_configure](https://www.cloudera.com/documentation/enterprise/5-6-x/topics/cdh_ig_hive_metastore_configure.html)

hive 自身，也需要数据库。

配置文件在 /etc/hive/conf/ 中。

