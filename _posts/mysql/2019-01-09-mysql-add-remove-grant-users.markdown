---
layout: post
title: "mysql 常用的用户相关操作"
date: 2019-01-09 21:40:00 +0800
categories: mysql
---

##### 创建用户
``` sql
CREATE USER 'username'@'host' IDENTIFIED BY 'password';
或者 insert into mysql.user(Host,User,Password) values("host","user",password("password"));
```

例子<br>
* 创建用户：insert into mysql.user(Host,User,Password) values("localhost","xiao004",password("xiao0004"));<br>
或者 CREATE USER 'xiao004'@'localhost' IDENTIFIED BY 'xiao0004';

这样就创建了一个名为 xiao004 密码为 xiao0004 的用户<br>
此处的 ***"localhost"***，是指该用户只能在本地登录，不能远程登录。如果想远程登录的话，将 "localhost" 改为 ***"%"***，表示在任何一台电脑上都可以登录。也可以指定某台机器可以远程登录

##### 用户授权
``` sql
GRANT privileges ON databasename.tablename TO 'username'@'host';
flush privileges;//授权后要刷新权限
```
**privileges** 表示权限类型
* all privileges：所有权限
* select：读取权限
* delete：删除权限
* update：更新权限
* create：创建权限
* drop：删除数据库、数据表权限

##### 创建用户同时授权
``` sql
1.grant all privileges on mq.* to test@localhost identified by '1234';
2.flush privileges;
```

##### 撤销用户授权
``` sql
REVOKE privilege ON databasename.tablename FROM 'username'@'host';
```
注意：假如你在给用户 'dog'@'localhost' 授权的时候是这样的(或类似的): GRANT SELECT ON test.user TO 'dog'@'localhost', 则在使用 REVOKE SELECT ON *.* FROM 'dog'@'localhost'; 命令并不能撤销该用户对 test 数据库中 user 表的 SELECT 操作. 相反, 如果授权使用的是 GRANT SELECT ON *.* TO 'dog'@'localhost'; 则 REVOKE SELECT ON test.user FROM 'dog'@'localhost'; 命令也不能撤销该用户对 test 数据库中 user 表的 Select 权限.<br>
***可以先通过 show grants for 用户名@主机; 命令查看指定用户的授权语句，然后再针对性的撤销权限***

##### 权限查询
* 查看 mysql 的所有用户及其权限：select * from mysql.user\G;
* 查看当前mysql用户权限：show grants;
* 查看某个用户的权限：show grants for 用户名@主机;

##### 对账号重命名
``` sql
rename user '旧用户名'@'旧主机' to '新用户名'@'新主机';
```

##### 修改用户密码
1. 使用 set password 命令：set password for '用户名'@'主机' = password('新密码');
2. 修改 mysql.user 表中的 password（或authentication_string）字段:<br>
update mysql.user set password=password('123123') where user='root' and host='localhost';<br>
注意：此方法一定要执行 ***“flush privileges;”*** 指令刷新权限，否则密码修改无法生效。Mysql5.7 以后应将 “password” 改为 “authentication_string”
3. 使用 grant 指令在授权时修改密码：<br>
grant select on 数据库名.数据表名 to 用户名@主机 identified by '新密码' with grant option;
4. 运行mysqladmin脚本文件: <br>
该文件一般在 mysql 安装目录下的 bin 目录中。进入该目录，根据一下两种具体情况输入命令（只有 root 用户有这个权限）
* 用户尚无密码：mysqladmin -u 用户名 password 新密码;
* 用户已有密码：mysqladmin -u 用户名 -p password 新密码;

##### 忘记密码登录 mysql
* 法一：先停止正在运行的 Mysql 服务，在命令行窗口进入 mysql 安装目录下的 bin 目录，在 -skip-grant-tables 参数下运行 mysqld 文件（Linux 系统运行 mysqld_safe文件更安全）：mysqld --skip-grant-tables<br>
这样可以跳过 Mysql 的访问控制，在控制台以管理员的身份进入 mysql 数据库。另外再开启一个命令行窗口，进入 mysql 安装目录下的 bin 目录，直接输入：mysql，回车，即可登录 mysql，然后就可以重新设置密码了（注意：此时 “Mysql 用户密码修改” 中的四种方法只有第二种方法能使用！）。设置成功后退出，重启 Mysql 服务
* 法二：修改 mysql 配置文件 my.ini。打开 mysql 配置文件 my.ini，在 '[mysqld]' 下加入 “skip-grant-tables”，保存，重启 Mysql 服务，然后就可以不需密码登录 mysql 进行密码修改了
