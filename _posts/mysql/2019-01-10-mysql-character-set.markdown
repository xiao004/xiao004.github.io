---
layout: post
title: "mysql character set"
date: 2019-01-10 13:00:00 +0800
categories: mysql
---

##### 查看 mysql 全局的编码设置
``` sql
show variables like 'character%';
show variables like 'collation%';
```

##### 修改 mysql 全局编码设置
1. 通过 set 命令修改，但是这种修改只能当前回话窗口生效，退出重新登录又要重新设置一遍
``` sql
#这些更改无法永久生效，只在当前会话中生效，当关闭客户端时就恢复成原来的编码了
set character_set_client=utf8
set character_set_connection=utf8
set character_set_database=utf8
set character_set_results=utf8
set character_set_server=utf8
```
2. 修改 my.ini 或者 my.cnf 文件。修改后重启 mysql 服务，永久生效
``` sql
[mysql]
default-character-set=utf8
[mysqld]
character_set_server=utf8 
collation_server=utf8_general_ci 
```

##### 查看数据库编码格式
``` sql
show variables like 'character_set_database';
```

##### 修改数据库的编码格式
``` sql
alter database <数据库名> character set utf8;
```

##### 查看表的编码格式
``` sql
show create table <表名>;
```

##### 修改数据表格的编码格式
``` sql
alter table <表名> character set utf8;
```

##### 创建数据库时指定数据库的字符集
``` sql
create database <数据库名> character set utf8;
```

##### 创建表时指定数据表的编码格式
``` sql
create table tb_books (
    name varchar(45) not null,
    price double not null,
    bookCount int not null,
    author varchar(45) not null ) default charset = utf8;
```

##### 查看字段编码格式
``` sql
SHOW FULL COLUMNS FROM tbl_name;
```

##### 修改字段编码格式
``` sql
alter table <表名> change <字段名> <字段名> <类型> character set utf8;
alter table user change username username varchar(20) character set utf8 not null;
```
