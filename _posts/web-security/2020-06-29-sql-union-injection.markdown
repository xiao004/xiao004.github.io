---
layout: post
title: "union sql injection"
date: 2020-06-29 22:30
categories: web-security
---

记一个 union sql 注入漏洞。union 操作符用于合并两个或多个 select 语句的结果集。注意，union 内部的 select 语句必须拥有相同数量的列，列也必须拥有相似的数据类型，同时每条 select 语句中的列的顺序必须相同。另外，默认情况下 union 操作符选取不同的值，如果想要查询重复的值需要用 union all

#### union sql 注入适用场景

1. 注入点页面有回显
2. union 连接的几个查询的字段数一样，且列的数据类型转换没有问题
3. 只有最后一个 select 子句允许有 order by 和 limit，即只有注入点前面的 sql 语句中能有 order by 和 limit 语句。select * from users order by id union select 1,2,3; 这种 sql 语句会报错

#### union sql 注入流程

1. 首先判断是否存在注入点及注入的类型
2. 观察回显的数据
3. 使用 order by 查询列数，order by x 即按照查询的 columns 中第 x 列排序，如果 x 大于查询的 columns 数量，则会报错
4. 获取数据库名——通过 database() 函数
5. 获取数据库中的表名——通过查 informaiton_schema.columns 表，参见 https://www.cnblogs.com/JiangLe/p/5793555.html
6. 获取数据库的表中的字段名
7. 获取字段中的数据

#### example——https://adworld.xctf.org.cn/task/answer?type=web&number=3&grade=1&id=4686&page=1

随便输入一个 a，发现下面有回显。盲猜一波可以 union 注入![union-sql-injection-1](/images/union-sql-injection-1.png)

使用 order by 语句确定查询的列数。输入 ' order by 4 #——这里的 ' 用于闭合前面的 ' 符号，# 用于注释掉后面的 '，发现页面 500 了，应该是查询的列数小于 4，换成 3 后查询成功。确定查询的列数为 3![union-sql-injection-2](/images/union-sql-injection-2.png)

尝试构造 union sql 注入语句获取数据库名——' union select 1, 2, database() #，得到数据库名 news。另外，还可以改进一下这个 sql 语句不显示前面语句的查询结果，可以猜到前面有 where 语句，因此在 union 前面加个 and 0 即可——' and 0 union select 1, 2, database() #![union-sql-injection-3](/images/union-sql-injection-3.png)

尝试获取数据表名。构造输入 ' and 0 union select 1, 2, table_name from information_schema.columns where table_schema='news' #，得到表名 secret_table

尝试获取 secret_table 表的列名。构造输入 ' and 0 union select 1, 2, column_name from information_schema.columns where table_name='secret_table' #，得到列名 id 和 fl4g

最后查询记录。构造输入 ' and 0 union select 1, id, fl4g from secret_table #，拿到了记录 id = 1, fl4g = QCTF{sq1_inJec7ion_ezzz}，即我们想要的 flag


