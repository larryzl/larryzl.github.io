---
layout: post
title: "MySQL 基础(一) InnoDB 事务隔离"
date: 2018-03-29 10:36:11 +0800
category: MySQL
tags: [MySQL]
---
* content
{:toc}


# 1. 事务介绍

MySQL 事务主要用于处理操作量大，复杂度高的操作。比如说，在人员管理系统中，你删除一个人员，你即需要删除人员资料，也需要删除人员相关的信息，如信箱、文章等等，这样，这些数据库操作语句就构成一个事务。

## 1.1. 事务满足4个条件（ACID）

- 原子性（Atomicity)：

	一个事务中的所有操作，要么全部执行，要么全部不执行，不会结束在中间某个环节。事务执行过程中发生错误，会被回滚到事务开始前的状态。
	
- 一致性（Consistency）： 

	在事务开始之前，和事务结束之后，数据库的完整性没有被破坏，这表示写入的资料必须完全符合所有的预设规则，这包含资料的精确度、串联性以及后续数据库可以自发性的完成预设的工作。
	
- 隔离性（Isolation）：

	数据库允许多个并发事务对其数据进行读写和修改的能力，隔离性可以防止多个事务并发执行时由于交叉执行而导致的数据不一致。事务隔离分为不同级别，包括:

	- 读未提交 Read uncommitted
	- 读提交 Read committed
	- 可重复读 repeatable read
	- 串行化 Serializable

- 持久性（Durability）：

	事务处理结束后，对数据的修改是永久的，即便系统故障也不会丢失。

## 1.2. 事务控制语句

在 MySQL 命令行的默认设置下，事务都是自动提交的，即执行 `SQL` 语句后就会马上执行 `COMMIT` 操作。因此要显式地开启一个事务务须使用命令 `BEGIN` 或 `START TRANSACTION`，或者执行命令 `SET AUTOCOMMIT=0`，用来禁止使用当前会话的自动提交。

- `BEGIN` 或 `START TRANSACTION`:	 

	显示的开启一个事务；
- `COMMIT`:	

	也可以使用`COMMIT WORK`，不过二者是等价的。`COMMIT`会提交事务，并使已对数据库进行的所有修改成为永久性的；
- `ROLLBACK`: 

	有可以使用`ROLLBACK WORK`，不过二者是等价的。回滚会结束用户的事务，并撤销正在进行的所有未提交的修改；
- `SAVEPOINT identifier`；

	`SAVEPOINT`允许在事务中创建一个保存点，一个事务中可以有多个`SAVEPOINT`；
- `RELEASE SAVEPOINT identifier`；

	删除一个事务的保存点，当没有指定的保存点时，执行该语句会抛出一个异常；
- `ROLLBACK TO identifier`；

	把事务回滚到标记点；
- `SET TRANSACTION`；

	用来设置事务的隔离级别。`InnoDB`存储引擎提供事务的隔离级别有`READ UNCOMMITTED`、`READ COMMITTED`、`REPEATABLE READ`和`SERIALIZABLE`。

## 1.3. MySQL 事务处理主要有两种方法

1. 用 BEGIN, ROLLBACK, COMMIT来实现

	- BEGIN 开始一个事务
	- ROLLBACK 事务回滚
	- COMMIT 事务确认

2. 直接用 SET 来改变 MySQL 的自动提交模式:

	- SET AUTOCOMMIT=0 禁止自动提交
	- SET AUTOCOMMIT=1 开启自动提交

# 2. 隔离性介绍

隔离性主要是指数据库系统提供一定的隔离机制，保证事务在不受外部并发操作影响的"独立"环境执行，意思就是多个事务并发执行时，一个事务的执行不应影响其它事务的执行。

## 2.1. 四种隔离级别

在SQL标准中定义了4种隔离级别，分别是：

- `Read uncommitted` 

	未提交读，事务中的修改，即使没有提交，对其他事务也是可见的。存在脏读

- `Read committed` 

	提交读，大多数数据库系统的默认隔离级别(MySQL不是), 一个事务从开始到提交之前，所做的修改对其他事务不可见。解决脏读，存在幻读和不可重复读

- `repeatable read`

	可重复读，该级别保证在同一事务中多次读取同样记录的结果是一致的。解决脏读和不可重复读，理论上存在幻读，但是在InnoDB引擎中解决了幻读

- `Serializable`

	可串行化，强制事务串行执行。

上面4种隔离级别是SQL标准定义的，但是在不同的存储引擎中，实现的隔离级别不尽相同。在InnoDB存储引擎中，`Repeatable Read` 是默认的事务隔离级别，同时该引擎的实现基于多版本的并发控制协议——`MVCC` (`Multi-Version Concurrency Control`)，解决了幻读问题，当然 脏读和不可重复读也是不存在的。`MVCC`最大的好处就在于读不加锁，读写不冲突，这样极大的增加了系统的并发性能

**幻读、脏读介绍：**

|类型|说明|
|---|---|
|幻读|指的是在同一事务下，连续执行两次同样的SQL语句可能导致不同的结果，第二次的SQL语句可能会返回之前不存在的行，也就是"幻行"。|
|脏读|一个事务A可以看到另一个事务B未提交的数据，如果此时事务B发生回滚，那么事务A拿到的就是脏数据，这就是脏读|
|不可重复读|个事务内根据同一条件对行记录进行多次查询，但是查询出的数据结果不一致，原因就是查询区间数据被其他事务修改了
|加锁读|

不可重复读感觉和幻读有点像，实际上，前者强调是同一行记录数据结果不一样，后者强调的时多次查询返回的结果集不一样，增加了或减少了。

## 2.2. Read uncommitted

未提交读，这种情况下，一个事务A可以看到另一个事务B未提交的数据，如果此时事务B发生回滚，那么事务A拿到的就是脏数据，这也就是脏读的含义。此隔离级别在MySQL InnoDB一般不会使用，不做过多说明。

## 2.3. Read Committed

提交读，一个事务从开始直到提交之前，所做的任何修改对其他事务都是不可见的。解决了脏读问题，但是存在幻读现象。

测试幻读：

1. 查看当前隔离级别

		SELECT @@tx_isolation;
		+-----------------+
		| @@tx_isolation  |
		+-----------------+
		| REPEATABLE-READ |
		+-----------------+

	设置隔离级别
	
		set session transaction isolation level read committed; 

2. 建表语句： 创建account表

		CREATE TABLE `account` (
		  `id` int(11) NOT NULL AUTO_INCREMENT,
		  `balance` int(6) NOT NULL,
		  `firstname` char(10) NOT NULL,
		  `lastname` char(10) NOT NULL,
		  `age` int(3) DEFAULT NULL,
		  `gender` char(2) DEFAULT NULL,
		  `address` char(30) DEFAULT NULL,
		  `city` char(10) DEFAULT NULL,
		  PRIMARY KEY (`id`)
		) ENGINE=InnoDB DEFAULT CHARSET=utf8;

3. 插入语句： 测试数据

		INSERT INTO account(id,balance,firstname,lastname,age) VALUES (1,222,"Amber","Duke",20);
		INSERT INTO account(id,balance,firstname,lastname,age) VALUES (2,233,"Hattie","Bond",20);
		INSERT INTO account(id,balance,firstname,lastname,age) VALUES (3,223,"Fulton","Holt",20);
		INSERT INTO account(id,balance,firstname,lastname,age) VALUES (4,323,"Hughes","Owens",21);

4. 开始测试

	![终端1](https://larryzl.github.io/images/blog/WX20180329-123005.png)

	![终端1](https://larryzl.github.io/images/blog/WX20180329-123157.png)

	步骤：
	
	1. 查看当前隔离级别
	2. 设置隔离级别 为 提交读
	3. 开启事务
	4. 第一次查询数据，表为空
	5. 在另一个 `client` 中插入数据
	6. 查询数据，发件可以查到数据

可以从上图看出，Read Committed这种隔离级别存在幻读现象。实际上，Read Committed还可能存在不可重复读的问题。


## 2.4. Repeatable Read

可重复读，该级别保证在同一事务中多次读取同样记录的结果是一致的，在InnoDB存储引擎中同时解决了幻读和不可重复读问题。至于InnoDB通过什么方式解决幻读和不可重复读问题，后续内容揭晓。

## 2.5. Serializable

Serializable 是最高的隔离级别，它通过 强制事务串行执行，避免了幻读的问题，但是 Serializable 会在读取的每一行数据上都加锁，所以可能导致大量的超时和锁争用的问题,因此并发度急剧下降，在MySQL InnoDB不被建议使用。

# 3. Read Committed 隔离级别下的加锁分析

隔离级别的实现与锁机制密不可分，所以需要引入锁的概念，首先我们看下InnoDB存储引擎提供的两种标准的行级锁：

- 共享锁 (S Lock)

	又称为读锁，可以允许多个事务并发的读取同一资源，互不干扰。即如果一个事务T对数据A加上共享锁后，其他事务只能对A再加共享锁，不能再加排他锁，只能读数据，不能修改数据

- 排它锁（X Lock)

	又称为写锁，如果事务T对数据A加上排他锁后，其他事务不能再对A加上任何类型的锁，获取排他锁的事务既能读数据，也能修改数据。

**注意**： 共享锁和排他锁是不相容的。

MySQL InnoDB 存储引擎是使用多版本并发控制的，**读不加锁，读写不冲突**，除非特定场景下的显示加读锁（这里不去探究）。本小节主要分析 `Read Committed` 隔离级别下的加锁情况，在MVCC的作用下，一般也就是写操作加X锁了。

加锁操作是和索引紧密相关的，对一个SQL语句进行加锁分析时，也要仔细考究其属性列上的索引类型。按照上文中标bank，有多个列，id列和其他列，插入了几条数据，没有明确索引情况：

	INSERT INTO account(id,balance,firstname,lastname,age) VALUES (1,222,"Amber","Duke",20);
	INSERT INTO account(id,balance,firstname,lastname,age) VALUES (2,233,"Hattie","Bond",20);
	INSERT INTO account(id,balance,firstname,lastname,age) VALUES (3,223,"Fulton","Holt",20);
	INSERT INTO account(id,balance,firstname,lastname,age) VALUES (4,323,"Hughes","Owens",21);
	
	
	


