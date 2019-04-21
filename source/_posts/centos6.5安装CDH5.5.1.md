---
title: centos6.5安装CDH5.5.1
date: 2016-9-29 11:52:18
---

### 安装并配置系统

#### 安装系统

系统使用 centos6.5 64位

[CentOS-6.5-x86_64-minimal](http://archive.kernel.org/centos-vault/6.5/isos/x86_64/CentOS-6.5-x86_64-minimal.iso)


**集群中使用到的所有主机必须使用完全相同的系统。**

#### 配置系统

1. 关闭 iptables

	``` bash
	# 关闭 iptables
	service iptables stop
	
	# 设置成开机不启动
	chkconfig iptables off
	```

2. 禁用 SELinux
	
	``` bash
	# method1
	vim /etc/selinux/config  # 改为 SELINUX=disabled
	
	# method2, 可以在脚本中使用
	sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
	
	# 临时禁用
	setenforce 0
	```

3. 设置 swappiness

	``` bash
	# 临时禁用
	echo 0 > /proc/sys/vm/swappiness
	# 或者
	sysctl vm.swappiness=10
	
	# 永久生效, 在下面文件最后
	# 添加 vm.swappiness=0 即可
	vim /etc/sysctl.conf
	```
	
4. 禁用 透明大页面
	
	``` bash
	# 临时生效
	echo never > /sys/kernel/mm/redhat_transparent_hugepage/defrag
	
	# 将下面的代码片断，添加到 /etc/rc.local 文件中。
	if test -f /sys/kernel/mm/redhat_transparent_hugepage/enabled; then
	echo never > /sys/kernel/mm/redhat_transparent_hugepage/enabled
	fi
	echo never > /sys/kernel/mm/redhat_transparent_hugepage/defrag
	```

#### 设置主机的 HOSTNAME

规划系统。准备4台机器，其中一台中安装clouderaManger 的 server, 其余3台机器作为 clouderaManger 的 agent. 假设，存在四台机器：192.168.0.1[0-3], 则执行以下配置：

1. 设置 HOSTNAME

	``` bash
	# 在下面文件中添加 HOSTNAME=<HOSTNAME>
	vi /etc/sysconfig/network
	```
	
	每台机器依次设置成如下 hostname:
	
	```
	192.168.0.10 <==> clouderaManger 
	192.168.0.11 <==> hadoop-worker1 
	192.168.0.12 <==> hadoop-worker2 
	192.168.0.13 <==> hadoop-worker3 
	```

2. 设置 hosts

	``` bash
	# 编辑 hosts 文件
	vi /etc/hosts
	```

	在所有主机上的 hosts 文件中追加如下配置:
	
	```
	192.168.0.10 clouderaManager
	192.168.0.11 hadoop-worker1
	192.168.0.12 hadoop-worker2
	192.168.0.13 hadoop-worker3
	```

上面的设置完成之后，需要重新启动系统。

**以下行文中出现 clouderManager主机 就表示 192.168.0.10 这台机器，同理 hadoop-worker1 表示 192.168.0.11 这台机器**

### 下载安装文件

提供，两种形式的包，parcel 和 yum


1. CouderaManager包

	parcel: 对于 CouderaManager 的 server 和 agent的安装，并没有提供 parcel 形式的包。
	
	yum: http://archive.cloudera.com/cm5/redhat/6/x86_64/cm/
	
	tarball: http://archive.cloudera.com/cm5/repo-as-tarball/

	需要下载的文件：
	
	[cloudera-manager-installer.bin](http://archive.cloudera.com/cm5/installer/5.5.1/)

	[clouderaManager-下载这个路径下的所有文件](http://archive.cloudera.com/cm5/redhat/6/x86_64/cm/5.5.1/RPMS/x86_64/)
	对于这个路径下的包，还有另一种下载方法，http://archive.cloudera.com/cm5/cm/5/cloudera-manager-el6-cm5.5.1_x86_64.tar.gz 这个路径下的就包含了上面文件夹中的所有包。直接下载即可。
	
	**将下载好的文件放置到 cm5.5.1 文件夹中。**

2. CDH 包

	parcel: http://archive.cloudera.com/cdh5/parcels/

	yum：http://archive.cloudera.com/cdh5/redhat/6/x86_64/cdh/
	
	tarball: http://archive.cloudera.com/cdh5/repo-as-tarball/

	需要下载的文件：
	如果使用 parcel 形式安装：则下载 http://archive.cloudera.com/cdh5/parcels/5.5.1/ 这个路径下的:
	
	1. [CDH-5.5.1-1.cdh5.5.1.p0.11-el6.parcel](http://archive.cloudera.com/cdh5/parcels/5.5.1/CDH-5.5.1-1.cdh5.5.1.p0.11-el6.parcel)
	
	2. [CDH-5.5.1-1.cdh5.5.1.p0.11-el6.parcel.sha1](http://archive.cloudera.com/cdh5/parcels/5.5.1/CDH-5.5.1-1.cdh5.5.1.p0.11-el6.parcel.sha1)
	
	3. [manifest.json](http://archive.cloudera.com/cdh5/parcels/5.5.1/manifest.json)
	
	如果使用 yum 方法安装则下载：
	
	http://archive.cloudera.com/cdh5/redhat/6/x86_64/cdh/5.5.1/RPMS/x86_64/
	
	这个路径下的所有rpm.
	
	将下载好的文件放置到 cdh5.5.1 文件夹中。

3. 安装所需的支持包，工具包

3.1 createrepo

3.2 postgresql

3.3 cloudera-manager-agent的依赖包

分别下载之。

最终将上面下载的文件，以如下的目录结构存放：

``` bash
+--- clouderaManager5.5.1
|   +--- cdh5.5.1
|   |   +--- CDH-5.5.1-1.cdh5.5.1.p0.11-el6.parcel
|   |   +--- CDH-5.5.1-1.cdh5.5.1.p0.11-el6.parcel.sha1
|   |   +--- manifest.json
|   +--- local-repo
|   |   +--- cm5.5.1
|   |   |   +--- cloudera-manager-agent-5.5.1-1.cm551.p0.8.el6.x86_64.rpm
|   |   |   +--- cloudera-manager-daemons-5.5.1-1.cm551.p0.8.el6.x86_64.rpm
|   |   |   +--- cloudera-manager-server-5.5.1-1.cm551.p0.8.el6.x86_64.rpm
|   |   |   +--- cloudera-manager-server-db-2-5.5.1-1.cm551.p0.8.el6.x86_64.rpm
|   |   |   +--- enterprise-debuginfo-5.5.1-1.cm551.p0.8.el6.x86_64.rpm
|   |   |   +--- jdk-6u31-linux-amd64.rpm
|   |   |   +--- oracle-j2sdk1.7-1.7.0+update67-1.x86_64.rpm
|   |   +--- createrepo
|   |   |   +--- createrepo-0.9.9-24.el6.noarch.rpm
|   |   +--- dependency
|   |   |   +--- agent
|   |   |   |   +--- apr-1.3.9-5.el6_2.x86_64.rpm
|   |   |   |   +--- ......
|   |   |   +--- agent-perl
|   |   |   |   +--- perl-5.10.1-141.el6_7.1.x86_64.rpm
|   |   |   |   +--- ......
|   |   |   +--- common
|   |   |   |   +--- wget-1.16.3-2.fc23.x86_64.rpm
|   |   +--- postgresql
|   |   |   +--- postgresql84-8.4.21-1PGDG.rhel6.x86_64.rpm
|   |   |   +--- postgresql84-contrib-8.4.21-1PGDG.rhel6.x86_64.rpm
|   |   |   +--- postgresql84-libs-8.4.21-1PGDG.rhel6.x86_64.rpm
|   |   |   +--- postgresql84-server-8.4.21-1PGDG.rhel6.x86_64.rpm
|   +--- cloudera-manager-installer.bin
```

使用命令进行打包：

``` bash
tar cvf clouderaManager5.5.1.tar clouderaManager5.5.1/
```

将 clouderaManager5.5.1.tar 上传到 clouderManager主机上。解压之：
``` bash
tar xvf clouderaManager5.5.1.tar
```

### 搭建本地 yum 源

#### 创建仓库

0. 安装 createrepo

1. 创建仓库所在文件夹

	``` bash
	mkdir -p /usr/local/local-repo/
	```

2. 将 clouderaManager5.5.1/local-repo 拷贝到 /usr/local/local-repo/

	``` bash
	cp clouderaManager5.5.1/local-repo /usr/local/local-repo/
	```

3. 创建仓库

	``` bash
	createrepo -v /usr/local/local-repo/
	```

4. 刷新仓库

	如果，有新的 rpm 添加进来，则使用下面的命令刷新。

	``` bash
	createrepo -v --update /usr/local/local-repo/
	```

5. 创建临时 yum 源服务器

	``` bash
	# 确定 8900 端口未被占用
	python -m SimpleHTTPServer 8900
	```
	
6. 验证

	使用 `http://clouderaManager:8900/` 访问yum仓库
	
	![](D:\cygwin64\home\Administrator\workspace\a-ray-of-sunshine.github.io\source\images\ClouderaManager-Yum.png)

#### 配置本地源

0. 创建 repo 文件

	在/etc/yum.repo.d/下创建一个local-repo.repo文件，添加如下内容到该文件
	
	``` bash
	[local-repo]                    #仓库名称可以自定义
	name=This is a local repo       #描述信息
	baseurl=http://192.168.0.1:8900 # 或者 http://clouderaManager:8900
	enabled=1                       #是否开启仓库，1为开启，0为关闭
	gpgcheck=0                      #是否检查gpgkey，1为开启，0为关闭
	```

1. 禁用外部源

	``` bash
	mv CentOS-Base.repo CentOS-Base.repo.bak
	```

2. 刷新yum缓存

	``` bash
	yum clean all
	```

3. 验证

	``` bash
	yum list | grep clouder*
	```
	
	可以看到本地仓库中rpm:
	
	cloudera-manager-agent.x86_64
	
	cloudera-manager-daemons.x86_64
	
	说明本地yum源安装成功。 

### 安装 Cloudera Manager Server

1. 进入到 clouderaManager5.5.1 目录

	``` bash
	cd ~/clouderaManager5.5.1/
	```

3. 安装 Server

	``` bash
	./cloudera-manager-installer.bin --skip_repo_package=1
	```
	
	**注意:已经配置好了本地仓库，所以添加--skip_repo_package=1，尤其是在clouderManager主机是未联网的情况下一定要添加这个选项**
	
	执行这个命令后会启动一个安装界面，按步骤安装即可。
	
	执行这个命令，会将安装日志记录到 `/var/log/cloudera-manager-installer/` 下面，如果安装过程中遇到问题，可以查看这里的日志信息。
	
	如果，正常安装，则可以看到以下日志记录：
	
	``` 
	0.check-selinux.log
	1.install-oracle-j2sdk1.7.log
	1.install-repo-pkg.log
	2.install-cloudera-manager-server.log
	2.install-oracle-j2sdk1.7.log
	3.install-cloudera-manager-server-db-2.log
	3.remove-cloudera-manager-repository.log
	4.start-embedded-db.log
	5.start-scm-server.log
	```
	
	可以看到如果安装完成，scm-server 将会启动。
	
4. 配置 CDH 源

	1. 拷贝 CDH 源
	
	将 clouderaManager5.5.1/cdh5.5.1 中所有的文件（3个）拷贝到 /opt/cloudera/parcel-repo 仓库中。
	
	``` bash
	cd ~/clouderaManager5.5.1/cdh5.5.1/
	cp * /opt/cloudera/parcel-repo/
	```

	2. 更改文件权限
	
	``` bash
	chown -R cloudera-scm:cloudera-scm ./*
	```
	
	Cloudera Manager Server 安装成功之后，会在 /opt 目录下创建 cloudera 文件夹，注意到cloudera-manager-installer.bin会创建一个名为 cloudera-scm 的用户和组，用 cloudera-scm 用户来安装配置文件等，所以配置的CDH源中的文件也需要将其改成 cloudera-scm。 
	
5. 验证

	在浏览器中输入 `http://<clouderaManager's IP>:7180/`, 应该会打开 Cloudera Manager 的 web 管理界面。默认的**用户名：admin, 密码：admin**
	
	如果没有打开，可以到 /var/log/cloudera-scm-server/ 目录中，查看日志
	
	``` bash
	tail -f cloudera-scm-server.log
	```
	
	**注意是 .log 文件，不是 .out 文件。**
	
	![](D:\cygwin64\home\Administrator\workspace\a-ray-of-sunshine.github.io\source\images\ClouderaManager-Login.png)

### 创建集群

#### 安装 agent

1. 为 CDH 群集安装指定主机

	![](D:\cygwin64\home\Administrator\workspace\a-ray-of-sunshine.github.io\source\images\ClouderaManager-FindHost.png)

	1. 输入主机或者ip，Cloudera Manger 将在这些主机上安装 agent,可以通过以下几种方式来指定主机。
	
		* 192.168.0.11, 192.168.0.12, 192.168.0.13
		* hadoop-worker1, hadoop-worker2, hadoop-worker3
		* 192.168.0.1[1-3]
		* hadoop-worker[1-3]
		
	2. 执行搜索
	
	![](D:\cygwin64\home\Administrator\workspace\a-ray-of-sunshine.github.io\source\images\ClouderaManager-CheckHost.png)
	
	选中需要安装的主机，点击继续
	
2. 群集安装

	1. 设置 Parcel 存储库，配置成本地源，将其指定为前面配置的临时yum源。http://clouderaManager:8900
	
	![](D:\cygwin64\home\Administrator\workspace\a-ray-of-sunshine.github.io\source\images\ClouderaManager-SetMediaSource.png)

	2. 设置 agent 的安装源，配置成本地源，将其指定为前面配置的临时yum源。http://clouderaManager:8900，然后继续
	
	![](D:\cygwin64\home\Administrator\workspace\a-ray-of-sunshine.github.io\source\images\ClouderaManager-SetAgentSource.png)
	
	3. JDK 安装选项
	
		勾选 安装 Oracle Java SE 开发工具包 (JDK)
		
	4. 正在安装
	
		![](D:\cygwin64\home\Administrator\workspace\a-ray-of-sunshine.github.io\source\images\ClouderaManager-InstallAgent.png)
		
		安装过程中，可以点击 详细信息 查看安装日志。
		
		也可以在 `/var/log/cloudera-scm-agent/` 中找到安装日志。 
	
		![](D:\cygwin64\home\Administrator\workspace\a-ray-of-sunshine.github.io\source\images\ClouderaManager-InstallSuccess.png)
		
		**这个步骤完成之后，agent就会启动，协助 server安装 CDH**
		
	5. 正在安装选定 Parcel
	
		![](D:\cygwin64\home\Administrator\workspace\a-ray-of-sunshine.github.io\source\images\ClouderaManager-InstallParcel.png)
		
	6. 检查主机正确性
	
		到此为到 agent 已经安装好了， CDH的 parcels包，也已经拷贝到 /opt/cloudera/parcels 处。CouderaManager server和agent安装完毕。

#### 群集设置


![](D:\cygwin64\home\Administrator\workspace\a-ray-of-sunshine.github.io\source\images\ClouderaManager-SelectService.png)
	
在这个界面中选择，需要安装的组件。安装默认配置安装即可。

安装配置完之后:

![](D:\cygwin64\home\Administrator\workspace\a-ray-of-sunshine.github.io\source\images\ClouderaManager-Success.png)

### 参考

* [CDH 5 and Cloudera Manager 5 Requirements and Supported Versions](http://www.cloudera.com/documentation/enterprise/release-notes/topics/rn_consolidated_pcm.html)
* [Cloudera-Manager-Installation-Guide](http://www.cloudera.com/documentation/manager/5-1-x/Cloudera-Manager-Installation-Guide/cm5ig_install_path_B.html)