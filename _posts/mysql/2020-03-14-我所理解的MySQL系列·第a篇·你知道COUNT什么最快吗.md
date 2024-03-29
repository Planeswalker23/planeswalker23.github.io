---
layout: post
title: 我所理解的MySQL系列·第a篇·你知道COUNT什么最快吗
categories: [MySQL]
keywords: MySQL, count
---



最近在学习极客时间丁奇的专栏《MySQL实战45讲》中第14讲有关 count 函数的时候觉得这一讲很有意思，遂决定加以实操并且记录，以加深印象。



![mysql-a-封面](https://cdn.jsdelivr.net/gh/Planeswalker23/image-storage@master/mysql/mysql-a-封面.3rp7j8fu3gc0.jpg)




## 1. 开篇词

最近在学习极客时间丁奇的专栏《MySQL实战45讲》中第14讲有关 count 函数的时候觉得这一讲很有意思，遂决定加以实操并且记录，以加深印象。

在 MySQL 中，当我们需要获取某张表中的总行数时，一般会选择使用下面的语句


```sql
select count(*) from table;
```


其实 count 函数中除了 * 还可以放其他参数，比如常数、主键 id 、字段，那么它们有什么区别？各自效率如何？我们应该使用哪种方式来获取表的行数呢？

当搞清楚 count 函数的运行原理后，相信上面几个问题的答案就会了然于胸。




## 2. 建表语句


为了解决上述的问题，我创建了一张 user 表，它有两个字段：主键 id 和 name ，后者可以为 NULL ，建表语句如下。


```sql
CREATE TABLE `user` (
  `id` int(11) NOT  NULL AUTO_INCREMENT COMMENT '主键',
  `name` varchar(255) DEFAULT  NULL COMMENT '姓名',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```


在该表中共有6000000条数据，前1000000条数据行的name字段为空，其余数据行 name=id，使用存储过程造测试数据的代码如下


```sql
-- 使用存储过程造测试数据
delimiter;;
create procedure idata()
begin 
  declare i int; 
  set i=1; 
  while(i<=6000000)do 
    insert into user values(i, i);
    set i=i+1; 
  end while;
end;;
delimiter;
call idata();
-- 将前1000000条数据的name字段置为 NULL
update user set name= NULL where id<1000000;
```



## 3. 执行 SQL 及结果


为了区分 count 函数不同参数的区别，主要从执行时间和扫描行数这两方面来描述 SQL 的执行效率，同时还会从返回结果来描述 count 函数的特性。


-  *符号 —— `select count(*) from user;`
- 常数—— `select count(1) from user;`
- 非空字段—— `select count(id) from user;`
- 可为空的字段—— `select count(name) from user;`



### 3.1 * 符号


```sql
mysql> select count(*) from user;
+----------+
| count(*) |
+----------+
|  6000000 |
+----------+
1 row in set (0.76 sec)
```


遍历全表，不取值（经优化器优化后，其内容必定不是 NULL，故不进行取值步骤），直接累加计数，最终返回结果。


### 3.2 常数


```sql
mysql> select count(1) from user;
+----------+
| count(1) |
+----------+
|  6000000 |
+----------+
1 row in set (0.76 sec)
```


遍历全表，一行行取数据，将每一行赋值为1，判断到该字段不可为空，累加计数，最终返回结果。


### 3.3 非空字段


```sql
mysql> select count(id) from user;
+-----------+
| count(id) |
+-----------+
|   6000000 |
+-----------+
1 row in set (0.85 sec)
```


遍历全表，一行行取数据（会选择最小的索引树来遍历，所以比相同情况下的 count 普通字段效率更高），取每行的主键 id ，判断到该字段不可为空，累加计数，最终返回结果。


### 3.4 可为空的字段


```sql
mysql> select count(name) from user;
+-------------+
| count(name) |
+-------------+
|     5900001 |
+-------------+
1 row in set (0.93 sec)
```


- 若字段定义不为空：遍历全表，一行行取数据，取每行的该字段，判断到该字段不可为空，累加计数，最终返回结果。
- 若字段定义可为空：遍历全表，一行行取数据，取每行的该字段，判断到该字段可能是 NULL ，所以需要再判断该字段的值到底是否为 NULL ，不为 NULL 才累加计数，最终返回结果。
- 若该字段没有索引，将遍历主键索引树，然后经过**回表**拿到整行数据，取得指定字段的值，再判断值是否为 NULL，不为 NULL 才累加计数，最终返回结果。



## 4. 执行结果分析


### 4.1 结果集


首先从结果集的角度来看，前三条 SQL 语句的目的是一样的——返回的是所有行数，而 count 函数的参数是可控字段的时候，它返回的是该字段不为 NULL 的行数。


### 4.2 执行时间


从执行时间上来看的话，执行时间大致是 **count(可为空的字段)** > **count(非空字段)** > **count(常数)** > **count(*)**。



## 5. 总结


 count 是一个聚合函数，对于返回的结果集需要一行行地判断值是否为空，如果 count 函数的参数不是 NULL ，累计值就加1，否则不加，最后返回累计值。


- **count(*)** 速度最快的原因是它不会在计数的时候去取每行数据值
-  **count(常数) ** 比  **count(*) ** 稍慢的原因是它会取每个数据行并赋值为1，多了一个赋值动作
-  **count(非空字段) ** 比  **count(常数) ** 稍慢的原因是它会从每个数据行中取出主键 id
-  **count(可为空的字段) ** 最慢的原因是它可能需要判断每个数据行中的改字段是否为 NULL

所以，最好还是用  **count(*) **。



## 6. 参考资料

- [极客时间-MySQL实战45讲](https://time.geekbang.org/column/intro/100020801)

最后，本文收录于个人语雀知识库: [我所理解的后端技术](https://www.yuque.com/planeswalker/bankend)，欢迎来访。