---
title: oracle-常用命令
date: 2016-11-2 9:19:30
---

1. sysdba 登录本地oracle.

	在忘记密码的时候，可以使用这个方法，登录到数据库安装的机器上然后，使用下面的命令登录 oracle 再使用 `alter` 重置密码。

	`sqlplus / as sysdba`

2. 导入数据

	`imp htm/htm file=htm.dmp full=y`

3. 在 windows 中启动 oracle 服务

	打开windows的服务管理界面，可以看到6个 Oracle 开关的服务，要想访问某个实例，则必须启动的服务是：
    
    * OracleServiceORCL

		启动数据库实例，其名称通常是： OracleService + sid

    * OracleOraDb11g_home1TNSListener

		启动 Listener， 此时数据库连接工具，就可以连接到 Listener，来访问上面的实例了。		

## PL/SQL Developer无法连接数据库

有时候出现，无法连接数据库，通过确认 PL/SQL Developer 使用 的 oci.dll 的版本来解决，

首先确认待连接的数据库版本，然后下载对应的版本的 OCI。就可以正常连接了。下载时还要注意下载 32 位，还是 64 位。

## 使用存储过程自动生成对表的 insert, update, select 语句

### 

``` bash
## 1. my_concat
create or replace function my_concat(tableName varchar2,type varchar2)
return varchar2
is
 type typ_cursor is ref cursor;
 v_cursor typ_cursor;
 v_temp varchar2(30);
 v_result varchar2(4000):= '';
 v_sql varchar2(200);
begin
 v_sql := 'select COLUMN_NAME from user_tab_columns where table_name = ''' || upper(tableName) || ''' order by COLUMN_ID asc';
 open v_cursor for v_sql;
 loop
    fetch v_cursor into v_temp;
    exit when v_cursor%notfound;
    if type = 'select' or type = 'insert' then
       v_result := v_result ||', ' || v_temp;
    elsif type = 'update' then
       v_result := v_result ||', ' || v_temp || ' = ?';
    elsif type = 'javabean' then
       v_result := v_result ||',bean.get' || upper(substr(v_temp,1,1)) || lower(substr(v_temp,2)) ||  '()';
    end if;
 end loop;
 return substr(v_result,2);
end;

## 2. 自动生成 SQL
create or replace procedure autoGenerateSQL(
tableName varchar2,
type varchar2,
out_result out varchar2
)
is
  sql_insert varchar2(2000);
  sql_update varchar2(2000);
  sql_select varchar2(2000);
  javabean_str varchar2(2000);
  field_num integer;        --字段个数
  type_info varchar2(20);   --参数类型判断信息
begin

sql_insert := 'INSERT INTO ' || upper(tableName) || '(' || my_concat(tableName,type) || ') VALUES (';
sql_update := 'UPDATE ' || upper(tableName) || ' SET ';
sql_select := 'SELECT ';
javabean_str := '';
type_info := '';

select count(*) into field_num from user_tab_columns where table_name=upper(tableName);
select decode(type,'insert',type,'update',type,'select',type,'javabean',type,'error') into type_info from dual;

if field_num = 0 then             -- 表不存在时
   out_result := '表不存在！请重新输入!';
elsif type_info = 'error' then    --type参数错误时
   out_result := 'type参数错误：参数必须小写，类型只能是insert、update、select、javabean之一';
elsif field_num > 0 then
  if type = 'insert' then         --生成insert 语句
    for i in 1..field_num
      loop
         sql_insert := sql_insert || '?';
         if i < field_num then
            sql_insert := sql_insert || ', ';
         end if;
      end loop;
      sql_insert := sql_insert || ')';
      out_result := sql_insert;
  elsif type = 'update' then      --生成update 语句
      sql_update := sql_update || my_concat(tableName,type);
      out_result := sql_update;
  elsif type = 'select' then      --生成select 语句
      sql_select := sql_select || my_concat(tableName,type) || ' FROM ' || upper(tableName) || ' A';
      out_result := sql_select;
  elsif type = 'javabean' then    --生成javabean的get方法
      javabean_str := my_concat(tableName,type);
      out_result := javabean_str;
  end if;
end if;

end autoGenerateSQL;
```

## 参考
1. [plsql中文乱码,显示问号](http://jingyan.baidu.com/article/a3aad71aa9bfefb1fa00964d.html)
2. [Oracle Instant Client Downloads](http://www.oracle.com/technetwork/database/features/instant-client/index-097480.html)
3. [Oracle-自动生成insert、update、select、javabean语句](http://blog.csdn.net/xiaomageit/article/details/52883920)