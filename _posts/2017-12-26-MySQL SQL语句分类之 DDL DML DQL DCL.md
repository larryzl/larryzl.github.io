---
layout: post
title: "MySQL SQL语句分类之 DDL DML DQL DCL"
date: 2017-12-26 16:09:44 +0800
category: MySQL
tags: [MySQL]
---
* content
{:toc}


## SQL语句分类

SQL 语言一共分为4类：


|名字|类型|作用的对象|作用|
|---|---|---|---|
|DDL（Data Definition Language）|数据定义语言|库、表、列|创建、删除、修改、库或表结构，对数据库或表的结构操作|
|DML（Data Manipulation Language）| 数据操作语言|数据库记录（数据）|增、删、改，对表记录进行更新|
|DQL（Data Query Language）| 数据查询语言|数据查询语言|查、用了查询数据，对标记录的查询|
|DCL（Data Control Language）| 数据控制语言|数据库用户|用来定义访问的权限和安全级别，对用户的创建，及授权|


## 数据定义语言DDL

>对象： 数据库和表

1. 对数据库的常用操作

|操作|命令|
|---|---|
|查看所有的数据库|`show databases;`|
|切换数据库|`use school;`|
|创建数据库|`create database mydb;`|
|删除数据库|`drop database mydb;`|
|修改数据库编码|`alter database mydb character set utf8;`|


2. 对标结构的常用操作

|操作|命令|
|---|---|
|创建表|`create table student;`|
|查看当前数据库的所有表名称|`show tables;`|
|查看指定某个表的创建语句|`show create table student;`|
|查看表结构|`desc student;`|
|删除表|`drop table student;`|
|修改表 添加列|`alter table student add password varchar(20);`|
|修改表 修改列类型|`alter table student modify password int;`|
|修改表 改列名|`alter table student change password pwd varchar(20);`|
|修改表 删除列|`alter table student drop pwd;`|
|修改表 修改表名|`alter table student rename teacher;`|


### MYSQL 5.7 在线DDL

1. MySQL各版本，对于DDL的处理方式是不同的，主要有三种：

	1. **Copy Table方式：** 

		这是InnoDB最早支持的方式。 顾名思义，通过临时表拷贝的方式实现的。新建一个带有新结构的临时表，将原表数据全部拷贝到临时表，然后Rename，完成创建操作。这个方式过程中，原表是可读的，不可写。但是会笑话一倍的存储空间。
		
	2. **Inplace方式：**

		这是原生MySQL 5.5 ，以及innodb_plugin中提供的方式。所谓Inplace，也就是在原表上直接进行，不会拷贝临时表。相对于Copy Table方式，这比较高效率。原表同意可读的，但是不可写。

	3. **Online方式：**

		这是MySQL 5.6以上版本中提供的方式。无论是Copy Table方式，还是Inplace方式，原表只能允许读取，不可写。 对应用有较大的限制，因此MySQL最新版本中，InnoDB支持了所谓的Online方式DDL。与以上两种方式相比，Online方式支持DDL时不仅可读，还可写。

2. MySQL 5.7 中 Online DDL：

	ALGORITHM=INPLACE，可以避免重建表代理的IO和CPU消耗，保证DDL期间依然有两盒的性能和并发。

	ALGORITHM=COPY，需要拷贝原始表，所以不允许并发DML写操作，可读。这种copy方式的效率还是不如Inplace，因为前者需要记录undo和redo log，而且因为临时占用buffer pool引起短暂时间内性能受影响。

	- In-Place为Yes是优选项，说明该操作支持INPLACE
	- Copies Table为No是优选项，因为为Yes需要重建表。大部分情况与In-Place是相反的
	- Allows Concurrent DML为Yes是优选项，说明DDL期间表一人可读写，可以指定LOCK=NONE（如果允许的话，mysql自动就是NONE）
	- Allows Concurrent Query 默认所有DDL操作期间都允许查询请求，放在这只是便于参考

## 数据操作语言DML

> 对象： 记录（行）
> 
> 关键词： `insert` `update` `delete`

1. 对表的操作

|操作|命令|
|---|---|
|插入所有字段|`insert into student values(01,'tonbby',99);`|
|插入指定字段|`insert into student(id,name) values(01,'tonbby');`|
|更新|`update student set name='tonbby',score=99 where id = 01;`|
|删除|`delete from student where id = 01;`|

## 数据查询语言DQL

> 对象： 记录
> 
> 关键词： `select`

语句：

	select * from student where id = 01;
	