---
title: Mysql安装
date: 2019-4-21 10:32:17
---

## 在 windows 上安装 mysql 

1. 下载地址

	[mysql 5.7 安装路径](https://dev.mysql.com/downloads/mysql/5.7.html#downloads)

2. 官网参考文档

	[Installing MySQL on Microsoft Windows Using a noinstall ZIP Archive](https://dev.mysql.com/doc/refman/5.7/en/windows-install-archive.html)
	
3. 安装过程

BASEDIR 为mysql 的安装路径。

* 解压到 C:\mysql
* 在 BASEDIR 下面创建mysql的配置文件。 my.ini
	
	增加如下配置。
	
	``` ini
	[mysqld]
	# set basedir to your installation path
	# basedir=c:\\mysql

	# set datadir to the location of your data directory
	# 最好就是安装到 basedir 下面，否则可能没有权限访问下面路径。导致Mysql服务启动失败。
	datadir=C:\\mysql\\data
	```
	
	注意配置 datadir 时，如果目录在后续访问中没有权限，在启动 server 的时候可能会报错。
	
* 初始化数据目录

	``` bash
	cd BASEDIR
	
	## 执行初始化任务
	### 方式1， 会生成随机密码
	bin\mysqld --initialize --console
	### 方式2， 不生成密码
	bin\mysqld --initialize-insecure --console
	```

	如果使用方式1：命令行会有如下提示：
	[Warning] A temporary password is generated for root@localhost:iTag*AfrH5ej

* 启动mysql 服务

	[Starting the Server for the First Time](https://dev.mysql.com/doc/refman/5.7/en/windows-server-first-start.html)
	
	`bin\mysqld --console`
	
* 登录
	
	1. If you used --initialize
		`mysql -u root -p` 使用这个命令，输入上面产生的随机密码。
	2. If you used --initialize-insecure
		`mysql -u root --skip-password` 使用这个命令登录
		
* 登录成功之后改密码

	`ALTER USER 'root'@'localhost' IDENTIFIED BY 'new_password';`

* 关闭 mysqld

	`bin\mysqladmin -u root -p shutdown`
	
* 安装 mysql 作为 windows 服务 

	!!注意提示 Install/Remove of the Service Denied! 启动cmd 时使用管理员权限!!
	
	!!然后跳转到 BASEDIR 下面执行命令!!
	
	!!不要使用环境变量中的 mysqld， 否则安装的服务会有问题!!


	``` bash
	// 需要手动启动。
	bin\mysqld --install-manual
	//  会随着系统启动自动启动。
	bin\mysqld --install
	
	## 删除上面安装的服务
	bin\mysqld --remove
	```
	
* 启动和关闭 mysql windows 服务

	!!如果启动失败，同理也要使用管理员权限执行命令!!

	``` bash
	## 启动 mysql 服务
	net start MySQL
	## 关闭 MySQL 服务
	net stop MySQL
	```
	
以管理员方式运行cmd, 在cmd上右键，以管理员方式运行即可。

## 关于索引的文档

[Indexes and Foreign Keys](https://dev.mysql.com/doc/refman/5.7/en/create-table.html#create-table-indexes-keys)
[CREATE INDEX Syntax](https://dev.mysql.com/doc/refman/5.7/en/create-index.html#create-index-unique)
[ALTER TABLE Syntax](https://dev.mysql.com/doc/refman/5.7/en/alter-table.html)
[How MySQL Uses Indexes](https://dev.mysql.com/doc/refman/5.7/en/mysql-indexes.html)
[Secondary Indexes and Generated Columns](https://dev.mysql.com/doc/refman/5.7/en/create-table-secondary-indexes.html)

## 参考
1. [从B 树、B+ 树、B* 树谈到R 树](https://blog.csdn.net/v_july_v/article/details/6530142)
2. [MySQL索引背后的数据结构及算法原理](http://blog.codinglabs.org/articles/theory-of-mysql-index.html)
3. [Optimization and Indexes](https://dev.mysql.com/doc/refman/5.7/en/optimization-indexes.html)
4. [Expression Syntax](https://dev.mysql.com/doc/refman/5.7/en/expressions.html)
5. []()
6. []()