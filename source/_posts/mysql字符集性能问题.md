---
title: mysql
date: 2016-6-15 11:30:14
tags: [mysql]
---

## mysql 相关操作

### 忘记密码

1. 在 my.ini 文件的 mysqld 下面添加skip-grant-tables，保存退出。
2. 重启 mysql, `service mysql restart`
3. 直接输入 `mysql` 就可以登录成功
4. 改密码

	``` java
	use mysql;
	update user set password=password("123456") where user="root";
	```

然后，使用新密码登录即可。如果不成功，重启一下mysql服务。

初次登录成功之后，可能出现，无法成功执行命令。会

### Mysql配置

可以在 option file 文件中进行配置。

[Using Option Files](http://dev.mysql.com/doc/refman/5.7/en/option-files.html)

1. mysqld

	[Server Option and Variable Reference](http://dev.mysql.com/doc/refman/5.7/en/mysqld-option-tables.html)
	
2. mysql

	[mysql Options](http://dev.mysql.com/doc/refman/5.7/en/mysql-command-options.html)

### 临时空间不够，无法进行查询

Got error 28 from storage engine

在硬盘空间大的地方，创建一个 tmp 目录。

``` bash
mkdir /<new_dir>/tmp
## 设置临时目录的权限为 dwrxwrxwrt
chmod 1777 /<new_dir>/tmp

## 编辑 my.cnf
## 在 [mysqld] 结点下添加一个配置
tmpdir=/<new_dir>/tmp
## 重启 mysql
service mysql restart

## 登录 mysql , 查看是否生效
SHOW VARIABLES LIKE 'tmp%';
```

## 显示不可见字符

select *, HEX(content) from tmp_table;

## $. 参考
1. [mysql账户和权限](http://dev.mysql.com/doc/refman/5.7/en/account-management-sql.html)
2. [FLUSH Syntax](http://dev.mysql.com/doc/refman/5.7/en/flush.html)
3. [http://dev.mysql.com/doc/refman/5.7/en/optimize-character.html](http://dev.mysql.com/doc/refman/5.7/en/optimize-character.html)
4. [http://dev.mysql.com/doc/refman/5.7/en/blob.html](http://dev.mysql.com/doc/refman/5.7/en/blob.html)
5. [skip-grant-tables](http://dev.mysql.com/doc/refman/5.7/en/server-options.html#option_mysqld_skip-grant-tables)
6. [How to Reset the Root Password](https://dev.mysql.com/doc/refman/5.7/en/resetting-permissions.html)
7. [Where MySQL Stores Temporary Files](https://dev.mysql.com/doc/refman/5.6/en/temporary-files.html)
8. [Errors, Error Codes, and Common Problems](https://dev.mysql.com/doc/refman/5.7/en/error-handling.html)