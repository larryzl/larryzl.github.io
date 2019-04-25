---
layout: post
title: "MySQL 备份整理(二） mysqldump介绍"
date: 2017-12-15 15:21:01 +0800
category: MySQL
tags: [MySQL]
---
* content
{:toc}


>mysqldump 是MySQL的一个命令行工具，用于逻辑备份。可以将数据库和表的结构，以及表中的数据分别导出成：create database, create table, insert into的sql语句。当然也可以导出 存储过程，触发器，函数，调度事件(events)。不管是程序员，还是DBA都会经常使用的一个工具。
>



### 1. 命令介绍

　　MySQL数据库自带的一个很好用的备份命令。是逻辑备份，导出 的是SQL语句。也就是把数据从MySQL库中以逻辑的SQL语句的形式直接输出或生成备份的文件的过程。

单实例语法：

	mysqldump -u <username> -p <dbname> > /path/to/***.sql
	
多实例语法：

	mysqldump -u <username> -p <dbname> -S <sockPath> > /path/to/***.sql
	

### 2. 参数介绍

	-A --all-databases：导出全部数据库
	-Y --all-tablespaces：导出全部表空间
	-y --no-tablespaces：不导出任何表空间信息
	--add-drop-database每个数据库创建之前添加drop数据库语句。
	--add-drop-table每个数据表创建之前添加drop数据表语句。(默认为打开状态，使用--skip-add-drop-table取消选项)
	--add-locks在每个表导出之前增加LOCK TABLES并且之后UNLOCK TABLE。(默认为打开状态，使用--skip-add-locks取消选项)
	--comments附加注释信息。默认为打开，可以用--skip-comments取消
	--compact导出更少的输出信息(用于调试)。去掉注释和头尾等结构。可以使用选项：--skip-add-drop-table --skip-add-locks --skip-comments --skip-disable-keys
	-c --complete-insert：使用完整的insert语句(包含列名称)。这么做能提高插入效率，但是可能会受到max_allowed_packet参数的影响而导致插入失败。
	-C --compress：在客户端和服务器之间启用压缩传递所有信息
	-B--databases：导出几个数据库。参数后面所有名字参量都被看作数据库名。
	--debug输出debug信息，用于调试。默认值为：d:t:o,/tmp/
	--debug-info输出调试信息并退出
	--default-character-set设置默认字符集，默认值为utf8
	--delayed-insert采用延时插入方式（INSERT DELAYED）导出数据
	-E--events：导出事件。
	--master-data：在备份文件中写入备份时的binlog文件，在恢复进，增量数据从这个文件之后的日志开始恢复。值为1时，binlog文件名和位置没有注释，为2时，则在备份文件中将binlog的文件名和位置进行注释
	--flush-logs开始导出之前刷新日志。请注意：假如一次导出多个数据库(使用选项--databases或者--all-databases)，将会逐个数据库刷新日志。除使用--lock-all-tables或者--master-data外。在这种情况下，日志将会被刷新一次，相应的所以表同时被锁定。因此，如果打算同时导出和刷新日志应该使用--lock-all-tables 或者--master-data 和--flush-logs。
	--flush-privileges在导出mysql数据库之后，发出一条FLUSH PRIVILEGES 语句。为了正确恢复，该选项应该用于导出mysql数据库和依赖mysql数据库数据的任何时候。
	--force在导出过程中忽略出现的SQL错误。
	-h --host：需要导出的主机信息
	--ignore-table不导出指定表。指定忽略多个表时，需要重复多次，每次一个表。每个表必须同时指定数据库和表名。例如：--ignore-table=database.table1 --ignore-table=database.table2 ……
	-x --lock-all-tables：提交请求锁定所有数据库中的所有表，以保证数据的一致性。这是一个全局读锁，并且自动关闭--single-transaction 和--lock-tables 选项。
	-l --lock-tables：开始导出前，锁定所有表。用READ LOCAL锁定表以允许MyISAM表并行插入。对于支持事务的表例如InnoDB和BDB，--single-transaction是一个更好的选择，因为它根本不需要锁定表。请注意当导出多个数据库时，--lock-tables分别为每个数据库锁定表。因此，该选项不能保证导出文件中的表在数据库之间的逻辑一致性。不同数据库表的导出状态可以完全不同。
	--single-transaction：适合innodb事务数据库的备份。保证备份的一致性，原理是设定本次会话的隔离级别为Repeatable read，来保证本次会话（也就是dump）时，不会看到其它会话已经提交了的数据。
	-F：刷新binlog，如果binlog打开了，-F参数会在备份时自动刷新binlog进行切换。
	-n --no-create-db：只导出数据，而不添加CREATE DATABASE 语句。
	-t --no-create-info：只导出数据，而不添加CREATE TABLE 语句。
	-d --no-data：不导出任何数据，只导出数据库表结构。
	-p --password：连接数据库密码
	-P --port：连接数据库端口号
	-u --user：指定连接的用户名。

### 3. 默认选项

mysqldump默认选项，有些是ture 有些是false，有的没有默认值。
默认为true的，具体含义如下：

	add-drop-table                    TRUE 表示在生成表结构语句之前，生成对应的 DROP TABLE IF EXISTS `table_name`; 语句
	add-locks                         TRUE 表示在生成表中数据的 insert into `table_name` values(...) 之前生成 LOCK TABLES `tab` WRITE;语句
	comments                          TRUE 表示生成备注，就是所有 -- 开头的说明，比如：-- Dumping data for  for table `tab`. 最好还是启用；
	create-options                    TRUE 表示在生成表结构时会生成：ENGINE=InnoDB AUTO_INCREMENT=827 DEFAULT CHARSET=utf8; 附加建表选项
	default-character-set             utf8 指定语句：/*!40101 SET NAMES utf8 */;中的字符集；可能你需要改成 --default-character-set=utf8mb4
	disable-keys                      TRUE 表示生产 insert 语句之前，生成：/*!40000 ALTER TABLE `tbl` DISABLE KEYS */; 可以加快insert速度；
	extended-insert                   TRUE 表示生产的insert是insert into `tbl` values(...),(...),数据行按照net-buffer-length分割合并成多个batch insert
	lock-tables                       TRUE 表示在导出的过程中会锁定所有表；
	max-allowed-packet                25165824 最大支持 24M 的数据包；
	net-buffer-length                 1046528  1M大小的socket buffer
	quick                             TRUE 表示在导出语句时，不缓存，直接输出到控制台或者文件中；
	quote-names                       TRUE 表示对表名和列名使用 `` 符号包裹；防止它们是关键字时会出错;
	set-charset                       TRUE default-character-set=utf8指定字符集，而--set-charset=1/0 表示是否生成/*!40101 SET NAMES utf8 */; 
	dump-date                         TRUE 表示是否在导出文件的末尾生成导出时间：-- Dump completed on 2015-09-15 11:15:10 
	secure-auth                       TRUE 表示登录判断密码时使用新的加密算法，拒绝就的加密算法
	triggers                          TRUE 表示生成触发器脚本；
	tz-utc                            TRUE 表示是否生成：/*!40103 SET @OLD_TIME_ZONE=@@TIME_ZONE */; /*!40103 SET TIME_ZONE='+00:00' */;


### 4. 登录服务器相关选项

1. 默认会顺序的从以下路径读取配置文件：

`/etc/my.cnf` `/etc/mysql/my.cnf` `/usr/local/mysql/etc/my.cnf` `~/.my.cnf`

2. 也可通过以下选项指定默认配置文件，那些 defaults 相关的选项都是为了另外指定 配置文件和登录文件，极少使用

		--no-defaults           Don't read default options from any option file, except for login file.
		--defaults-file=#       Only read default options from the given file #.
		--defaults-extra-file=# Read this file after the global files are read.
		--defaults-group-suffix=# Also read groups with concat(group, suffix)
		--login-path=#          Read this path from the login file.

3. 这几个选项指定 登录的用户名，密码，mysqld IP地址，端口，连接使用的协议等等。

		-u, --user=name     User for login if not current user.
		-p, --password[=name] Password to use when connecting to server. If password is not given it's solicited on the tty.		
		-h, --host=name     Connect to host.	
		-P, --port=#        Port number to use for connection.
		--protocol=name     The protocol to use for connection (tcp, socket, pipe, memory)
		--max-allowed-packet=#  The maximum packet length to send to or receive from server.
		--net-buffer-length=#      The buffer size for TCP/IP and socket communication.


	般常用的是 -h192.168.2.xx -uxxx -p ，如果mysqld默认端口不是3306，则需要使用 -Pxxx 指定端口.

	--max-allowed-packet 我们一般配置在my.cnf中。--net-buffer-length 是为了优化网络连接的socket buffer.

	使用示例： `mysqldump -h192.168.1.20 -uxxx -p -P3057`
	
### 5. 备份内容相关选项

我们可以选择备份所有数据库，某几个数据库，某一个数据库，某一个数据库中的某几个表，某一个数据库中的一个表；

可以选择是否备份 存储过程和函数，触发器，调度事件.

1. 选择导出的数据库 和 表：

		-A, --all-databases Dump all the databases. This will be same as --databases with all databases selected.
		
		-B, --databases     Dump several databases. Note the difference in usage; in this case no tables are given. All name arguments are regarded as database names. 'USE db_name;' will be included in the output.

2. 选择是否导出 建库，建表语句，是否导出 表中的数据：

		-n, --no-create-db  Suppress the CREATE DATABASE ... IF NOT EXISTS statement that normally is output for each dumped database if --all-databases or --databases is given. (不导出建库语句: CREATE DATABASE，也就是不导库结构)	
		-t, --no-create-info Don't write table creation info. (不导出建表语句)		
		-d, --no-data         No row information. (不导出数据，有时我们仅仅需要导出表结构，也就是建表语句就行了)
		
3. 选择是否导出 存储过程和函数，触发器，调度事件：

		-R, --routines      Dump stored routines (functions and procedures). (导出存储过程和函数)
		--triggers            Dump triggers for each dumped table. (Defaults to on; use --skip-triggers to disable.) (导出触发器)
		--skip-triggers     不导出触发器
		-E, --events        Dump events. 导出调度事件(根据备份的目的进行选择，如果是搭建slave，那么就不要导出events.)

4. 指定不导出 某个库的某个表：

		--ignore-table=name   Do not dump the specified table. To specify more than one table to ignore, use the directive multiple times,
		
		                                 once for each table.  Each table must be specified with both database and table names,
		
		                                 e.g.,  --ignore-table=database.table. (在导出数据库时，排除某个或者某几个表不导出)

5. 按照where条件导出

		-w, --where='where_condition' Dump only selected records. Quotes are mandatory.
		
6. 使用实例

	|说明|命令|备注|
	|---|---|----|
	|导出单表的结构和数据|`mysqldump -uxxx -p db1 tb1 > tb1.sql`|导出数据库db1中的表tb1的表结构与数据|
	|导出多表的结构和数据|`mysqldump -uxxx -p db1 tb1 tb2 > tb1_tb2.sql`|导出数据库db1中的表tb1、tb2的表结构与数据|
	|导出单表的表结构|`mysqldump -uxxx -p --no-data db1 tb1 > tb1.sql`|导出数据库db1中的表tb1的表结构<br>导出内容与`show create table tb1`相同|
	|我们无法使用mysqldump达到只导出某个或某几个表的数据，而不导出建表语句的目的。但是我们可以使用`select * from table into outfile 'file.sql'`，比如 `select * from Users into outfile '/tmp/Users.sql';`，注意需要对目录有写权限|
	|导出单个库中库结构、表结构、表数据|`mysqldump -uxxx -p --databases db1 >db1.sql`||
	|导出多个库中的库结构、表结构、表数据|`mysqldump -uxxx -p --databases db1 db2 >db1_db2.sql`||
	|导出单个库中库结构、表结构、不需要数据|`mysqldump -uxxx -p --no-data --databases db1 > db1.sql`||
	|导出单个库中数据，不需要库结构和表结构|`mydsqldump -uxxx -p --no-create-db --no-create-info --databases db1 > db1.sql`||
	|导出多个库中库结构，表结构、不需要表数据|`mysqldump -uxxx -p --no-data --databases db1 db2 > db1_db2.sql`||
	|导出数据库中所有库的库结构，表结构，数据|`mysqldump -uxxx -p --all-databases > all.sql`||
	|导出数据库中所有库的库结构、表结构，不需要数据|`mysqldump -uxxx -p --all-databases --no-data > all.sql`||
	|导出单个库中库结构、表结构、表数据、排除某个表|`mysqldump -uxxx -p --databases db1 --ignore-table=db1.test >db1.sql`||

### 6. mysqldump事务和数据一致性（锁）相关

在使用mysqldump 逻辑备份时，事务和数据一致性的选项是至关重要的。

1.  --single-transaction

	--single-transaction 可以得到一致性的导出结果。他是通过将导出行为放入一个事务中达到目的的。
	
	它有一些要求：只能是 innodb 引擎；导出的过程中，不能有任何人执行 alter table, drop table, rename table, truncate table等DDL语句。
	
	实际上DDL会被事务所阻塞，因为事务持有表的metadata lock 的共享锁，而DDL会申请metadata lock的互斥锁，所以阻塞了。
	
	--single-transaction 会自动关闭 --lock-tables 选项；上面我们说到mysqldump默认会打开了--lock-tables，它会在导出过程中锁定所有表。
	
	因为 --single-transaction 会自动关闭--lock-tables，所以单独使用--single-transaction是不会使用锁的。与 --master-data 合用才有锁。

2. --lock-tables

	该选项默认打开的。作用是在导出过程中锁定所有表。 --single-transaction 和--lock-all-tables 都会将该选项关闭
	
3. --lock--all-tables

	在备份过程中增加全局读锁，启动该选项，会自动关闭 --single-transaction 和 --lock-tabl
	
	上面三个选项中，只有 --lock-tables 默认是打开的；打开 --single-transaction 或者 打开 --lock-all-tables 都将关闭 --lock-tables. 而--lock-all-tables会自动关闭 --single-transaction 和 --lock-tables。所以三者是互斥的。我们应该一次只启用其中一个选项。

4. --flush-logs

	为了获得导出数据和刷新日志的一致性(同时发生)，必须将 --flush-logs 选项和 --lock-all-tables 或者 --master-data 一起使用：
	
		mysqldump --flush-logs --lock-all-tables;  mysqldump --flush-logs --master-data=2 ；
		
5. --flush-privileges 

	如果导出包含了mysql数据，就应该启用该选项。该选项会在导出的 mysql 数据库的后面加上 flush privileges 语句，因为在向mysql数据库inert了语句之后，必须使用 flush privileges，不然权限不生效。
	
6. --master-data

	为了获得一致性的备份数据和在备份时同时刷新binary日志，我们应该如下结合使用这些选项
	
		mysqldump -uxxx -pxxx --single-transaction --master-data=2 --flush-logs --routines --databases db1 > db1.sql;
		#其中 --flush-logs 不是必须的; 搭建slave时，不要导出events，但是需要导出rountines.
		
	其中被 --master-data 打开的 --lock-all-tables 选项，又被 --single-transaction 关闭掉了。--flush-logs 借助于 --master-data 可以达到即使一次导出多个数据库时，其 flush 的二进制日志也是在同一个时间点的，不是每一个数据库flush一次的。并且这个时间点 和 --master-data 记录的 binary log position 和 binary log file是同一个时间点，这些都是利用了 --single-transaction 和 --master-data 合用时短暂的使用一个全局的读锁来达到目的的。


### 7. mysqldump 复制的相关选项

1. --master-data

	该选项，上面已经介绍。--master-data=1表示会导出 change master to语句，--master-data=2 该语句放在注释中，默认是0 。 一般会和 --single-transaction一起使用，用于搭建master-slave 环境。
	
		mysqldump -uxxx -p --master-data=1 --databases db1 > db1.sql
		
		# db1.sql 内容
		
		CHANGE MASTER TO MASTER_LOG_FILE='mysql-bin.000002', MASTER_LOG_POS=369709;
		
		mysqldump -uxxx -p --master-data=2 --databases db1 > db1.sql
		
		# db1.sql 内容
		--
		-- Position to start replication or point-in-time recovery from
		--
		
		-- CHANGE MASTER TO MASTER_LOG_FILE='mysql-bin.000002', MASTER_LOG_POS=369709;
		
		--

2. `--dump-slave[=#]`

      --dump-slave 和 --master-data 几乎一样。区别只是--dump-slave用于slave建立下一级的slave；而 --master-data用于master建立slave;

      如果在 master 上使用 --dump-slave 会报错：mysqldump: Couldn't execute 'START SLAVE': The server is not configured as slave;
      
3. `--apply-slave-statements`

  在 change master 导出 stop slave 语句, 在 change master 之后导出 start slave语句。其实是一个自动化的处理。和 --master-data=1 类似。


4.  `--include-master-host-port `

	在`CHANGE MASTER TO ..`处添加`MASTER_HOST=<host>,MASTER_PORT=<port>` ，该选择要结合`--dump-slave=1/2`
	
5. `--delete-master-logs `

	备份之后删除master上的binary log。该选项会自动打开`--master-data=2`，该选项一般不使用，日本一般不能随便删除
	
6. `--set-gtid-purged[=name]`

	Add 'SET @@GLOBAL.GTID_PURGED' to the output. Possible
	  values for this option are ON, OFF and AUTO. If ON is
	  used and GTIDs are not enabled on the server, an error is
	  generated. If OFF is used, this option does nothing. If
	  AUTO is used and GTIDs are enabled on the server, 'SET
	  @@GLOBAL.GTID_PURGED' is added to the output. If GTIDs
	  are disabled, AUTO does nothing. If no value is supplied
	  then the default (AUTO) value will be considered.
	  
	 该选项用于启用了GTID特性的环境。
	 
### 8. mysqldump 字符集的相关选项

1. `--set-charset`

	Add 'SET NAMES default_character_set' to the output.
   (Defaults to on; use --skip-set-charset to disable.)
   `--set-charset=1/0`开启和关闭。也可以使用`--skip-set-charset`关闭
   该选项表示是否生成`/*!40101 SET NAMES utf8 */;`
   
2. `-N, --no-set-names`

	关闭`--set-charset`

3. `--default-character-set=name`

	指定语句：`/*!40101 SET NAMES utf8 */;`中的字符集；可能你需要改成 --default-character-set=utf8mb4
	
### 9. mysqldump 控制是否生成DDL语句的相关选项

1. --add-drop-database  Add a DROP DATABASE before each create.
2. --add-drop-table        Add a DROP TABLE before each create.  (Defaults to on; use --skip-add-drop-table to disable.)
3. --add-drop-trigger      Add a DROP TRIGGER before each create.
4. --no-create-db,-n 
5. --no-create-info,-t


### 10. mysqldump导出格式相关选项

1. `--compatible=name`

	导出sql语句的兼容模式。如果我们需要从MySQL导出，然后导入到其他数据库，则可使用该选项。
	
2. `-Q,--quote-names`

	将表名喝列名使用\`\`包裹。以防他们是关键字时报错。
	
### 	11. mysqldump 错误处理的相关选项
1. `-f,--force`
	
	遇到错误继续执行

2. `--log-error=name`

	将warnings 和eerors存到文件
	
### 12. mysqldump实现原理

为了探求 mysqldump 的备份是如何实现的，我们需要在 my.cnf 中的[mysqld] 参数段加入：

	general_log = on
	general_log_file = general.log
	
这样我们就可以通过观察general.log的输出，来了解mysqldump的备份是如何实现的。

1. `--lock-tables`

	先执行：mysqldump -uroot -p --databases gs --lock-tables > gs_l.sql， 然后查看 general.log:

		 20 Init DB gs
		 20 Query SHOW CREATE DATABASE IF NOT EXISTS `gs`
		 20 Query show tables
		 20 Query LOCK TABLES `tb1` READ /*!32311 LOCAL */,`user` READ /*!32311 LOCAL */
		 20 Query show table status like 'tb1'
		 20 Query SET SQL_QUOTE_SHOW_CREATE=1
		 20 Query SET SESSION character_set_results = 'binary'
		 20 Query show create table `tb1`
		 20 Query SET SESSION character_set_results = 'utf8'
		 20 Query show fields from `tb1`
		 20 Query show fields from `tb1`
		 20 Query SELECT /*!40001 SQL_NO_CACHE */ * FROM `tb1`
		 20 Query SET SESSION character_set_results = 'binary'
		 20 Query use `gs`
		 20 Query select @@collation_database
		 20 Query SHOW TRIGGERS LIKE 'tb1'
		 20 Query SET SESSION character_set_results = 'utf8'
		 20 Query show table status like 'user'
		 20 Query SET SQL_QUOTE_SHOW_CREATE=1
		 20 Query SET SESSION character_set_results = 'binary'
		 20 Query show create table `user`
		 20 Query SET SESSION character_set_results = 'utf8'
		 20 Query show fields from `user`
		 20 Query show fields from `user`
		 20 Query SELECT /*!40001 SQL_NO_CACHE */ * FROM `user`
		 20 Query SET SESSION character_set_results = 'binary'
		 20 Query use `gs`
		 20 Query select @@collation_database
		 20 Query SHOW TRIGGERS LIKE 'user'
		 20 Query SET SESSION character_set_results = 'utf8'
		 20 Query UNLOCK TABLES
		 20 Quit

	
	日志分析：
	
	1.  ```SHOW CREATE DATABASE IF NOT EXISTS `gs````  导出建库语句；
	2.  `show tables` 活的数据库中所有表名，然后锁住：`LOCK TABLES `tb1` READ /*!32311 LOCAL */,`user` READ /*!32311 LOCAL */`
	3.  `show create table 'tb1'` 导出tb1建表语句
	4.  ```show fields from `tb1```` 导出了表中的数据；
	
		。。。。
	5.  最后导出了trigger，然后unlock tables，结束

		可以看到 --lock-tables 在导出一个数据库时，会在整个导出过程 lock read local 所有的表。该锁不会阻止其它session读和插入。
		
2. `--lock-all-tables`

	先执行：mysqldump -uroot -p --databases gs --lock-all-tables > gs_l.sql， 在查看 general.log:

		 21 Connect root@localhost on using Socket
		 21 Query /*!40100 SET @@SQL_MODE='' */
		 21 Query /*!40103 SET TIME_ZONE='+00:00' */
		 21 Query FLUSH TABLES
		 21 Query FLUSH TABLES WITH READ LOCK
		 21 Query SHOW VARIABLES LIKE 'gtid\_mode'
		 21 Query SELECT @@GLOBAL.GTID_EXECUTED
		 21 Query SELECT LOGFILE_GROUP_NAME, FILE_NAME, TOTAL_EXTENTS, INITIAL_SIZE, ENGINE, EXTRA FROM INFORMATION_SCHEMA.FILES WHERE FILE_TYPE = 'UNDO LOG' AND FILE_NAME IS NOT NULL AND LOGFILE_GROUP_NAME IS NOT NULL AND LOGFILE_GROUP_NAME IN (SELECT DISTINCT LOGFILE_GROUP_NAME FROM INFORMATION_SCHEMA.FILES WHERE FILE_TYPE = 'DATAFILE' AND TABLESPACE_NAME IN (SELECT DISTINCT TABLESPACE_NAME FROM INFORMATION_SCHEMA.PARTITIONS WHERE TABLE_SCHEMA IN ('gs'))) GROUP BY LOGFILE_GROUP_NAME, FILE_NAME, ENGINE, TOTAL_EXTENTS, INITIAL_SIZE ORDER BY LOGFILE_GROUP_NAME
		 21 Query SELECT DISTINCT TABLESPACE_NAME, FILE_NAME, LOGFILE_GROUP_NAME, EXTENT_SIZE, INITIAL_SIZE, ENGINE FROM INFORMATION_SCHEMA.FILES WHERE FILE_TYPE = 'DATAFILE' AND TABLESPACE_NAME IN (SELECT DISTINCT TABLESPACE_NAME FROM INFORMATION_SCHEMA.PARTITIONS WHERE TABLE_SCHEMA IN ('gs')) ORDER BY TABLESPACE_NAME, LOGFILE_GROUP_NAME
		 21 Query SHOW VARIABLES LIKE 'ndbinfo\_version'
		 21 Init DB gs
		 21 Query SHOW CREATE DATABASE IF NOT EXISTS `gs`
		 21 Query show tables
		 21 Query show table status like 'tb1'
		 21 Query SET SQL_QUOTE_SHOW_CREATE=1
		 21 Query SET SESSION character_set_results = 'binary'
		 21 Query show create table `tb1`
		 21 Query SET SESSION character_set_results = 'utf8'
		 21 Query show fields from `tb1`
		 21 Query show fields from `tb1`
		 21 Query SELECT /*!40001 SQL_NO_CACHE */ * FROM `tb1`
		 21 Query SET SESSION character_set_results = 'binary'
		 21 Query use `gs`
		 21 Query select @@collation_database
		 21 Query SHOW TRIGGERS LIKE 'tb1'
		 21 Query SET SESSION character_set_results = 'utf8'
		 21 Query show table status like 'user'
		 21 Query SET SQL_QUOTE_SHOW_CREATE=1
		 21 Query SET SESSION character_set_results = 'binary'
		 21 Query show create table `user`
		 21 Query SET SESSION character_set_results = 'utf8'
		 21 Query show fields from `user`
		 21 Query show fields from `user`
		 21 Query SELECT /*!40001 SQL_NO_CACHE */ * FROM `user`
		 21 Query SET SESSION character_set_results = 'binary'
		 21 Query use `gs`
		 21 Query select @@collation_database
		 21 Query SHOW TRIGGERS LIKE 'user'
		 21 Query SET SESSION character_set_results = 'utf8'
		 21 Quit
				 

	它的实现使用了 FLUSH TABLES; FLUSH TABLES WITH READ LOCK; 语句。在最后没有看到解锁语句。

	它请求发起一个全局的读锁，会阻止对所有表的写入操作，以此来确保数据的一致性。备份完成后，该会话断开，会自动解锁。

3. `--single-transaction `

	先执行： mysqldump -uroot -p --databases gs --single-transaction > gs_l.sql，在查看 general.log:
	
	
		 22 Connect root@localhost on using Socket
		 22 Query /*!40100 SET @@SQL_MODE='' */
		 22 Query /*!40103 SET TIME_ZONE='+00:00' */
		 22 Query SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ
		 22 Query START TRANSACTION /*!40100 WITH CONSISTENT SNAPSHOT */
		 22 Query SHOW VARIABLES LIKE 'gtid\_mode'
		 22 Query SELECT @@GLOBAL.GTID_EXECUTED
		 22 Query UNLOCK TABLES
		 22 Query SELECT LOGFILE_GROUP_NAME, FILE_NAME, TOTAL_EXTENTS, INITIAL_SIZE, ENGINE, EXTRA FROM INFORMATION_SCHEMA.FILES WHERE FILE_TYPE = 'UNDO LOG' AND FILE_NAME IS NOT NULL AND LOGFILE_GROUP_NAME IS NOT NULL AND LOGFILE_GROUP_NAME IN (SELECT DISTINCT LOGFILE_GROUP_NAME FROM INFORMATION_SCHEMA.FILES WHERE FILE_TYPE = 'DATAFILE' AND TABLESPACE_NAME IN (SELECT DISTINCT TABLESPACE_NAME FROM INFORMATION_SCHEMA.PARTITIONS WHERE TABLE_SCHEMA IN ('gs'))) GROUP BY LOGFILE_GROUP_NAME, FILE_NAME, ENGINE, TOTAL_EXTENTS, INITIAL_SIZE ORDER BY LOGFILE_GROUP_NAME
		 22 Query SELECT DISTINCT TABLESPACE_NAME, FILE_NAME, LOGFILE_GROUP_NAME, EXTENT_SIZE, INITIAL_SIZE, ENGINE FROM INFORMATION_SCHEMA.FILES WHERE FILE_TYPE = 'DATAFILE' AND TABLESPACE_NAME IN (SELECT DISTINCT TABLESPACE_NAME FROM INFORMATION_SCHEMA.PARTITIONS WHERE TABLE_SCHEMA IN ('gs')) ORDER BY TABLESPACE_NAME, LOGFILE_GROUP_NAME
		 22 Query SHOW VARIABLES LIKE 'ndbinfo\_version'
		 22 Init DB gs
		 22 Query SHOW CREATE DATABASE IF NOT EXISTS `gs`
		 22 Query SAVEPOINT sp
		 22 Query show tables
		 22 Query show table status like 'tb1'
		 22 Query SET SQL_QUOTE_SHOW_CREATE=1
		 22 Query SET SESSION character_set_results = 'binary'
		 22 Query show create table `tb1`
		 22 Query SET SESSION character_set_results = 'utf8'
		 22 Query show fields from `tb1`
		 22 Query show fields from `tb1`
		 22 Query SELECT /*!40001 SQL_NO_CACHE */ * FROM `tb1`
		 22 Query SET SESSION character_set_results = 'binary'
		 22 Query use `gs`
		 22 Query select @@collation_database
		 22 Query SHOW TRIGGERS LIKE 'tb1'
		 22 Query SET SESSION character_set_results = 'utf8'
		 22 Query ROLLBACK TO SAVEPOINT sp
		 22 Query show table status like 'user'
		 22 Query SET SQL_QUOTE_SHOW_CREATE=1
		 22 Query SET SESSION character_set_results = 'binary'
		 22 Query show create table `user`
		 22 Query SET SESSION character_set_results = 'utf8'
		 22 Query show fields from `user`
		 22 Query show fields from `user`
		 22 Query SELECT /*!40001 SQL_NO_CACHE */ * FROM `user`
		 22 Query SET SESSION character_set_results = 'binary'
		 22 Query use `gs`
		 22 Query select @@collation_database
		 22 Query SHOW TRIGGERS LIKE 'user'
		 22 Query SET SESSION character_set_results = 'utf8'
		 22 Query ROLLBACK TO SAVEPOINT sp
		 22 Query RELEASE SAVEPOINT sp
		 22 Quit
		 
	 基本过程：

	1. 先改变事务隔离级别：SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ

	2. 开始事务：START TRANSACTION /*!40100 WITH CONSISTENT SNAPSHOT */

	3. unlock tables;
	
	4. 导出建库语句； SHOW CREATE DATABASE IF NOT EXISTS `gs`
	
	5. 打开一个 savepoint: SAVEPOINT sp;
	
	6. 导出 表 tb1 的结构和数据；
	
	7. ROLLBACK TO SAVEPOINT sp; 回滚到savepoint;
	
		对其它表重复该过程；
	
	8. 最后 realease savepoint p; 释放savepoint;
	
	整个过程，没有任何锁。RR隔离级别保证在事务中只读取本事务之前的一致性的数据。 rollback to savepoint sp; 保证了对数据库中的数据没有影响。
	

### 12. mysqldump 与锁

1）--lock-tables 会在整个导出过程 lock read local 所有的表。该锁不会阻止其它session读和插入，但是显然阻塞了update。

2）--lock-all-tables 它请求发起一个全局的读锁，会阻止对所有表的写入操作(insert,update,delete)，以此来确保数据的一致性。备份完成后，该会话断开，会自动解锁。

3）--single-transaction 和 --master-data 结合使用时，也是在开始时，会短暂的请求一个全局的读锁，会阻止对所有表的写入操作。

4）--single-transaction 单独使用，不会有任何锁。但是测试表明: 它也需要对备份的表持有 metadata lock 的共享锁。

而我们知道，一般的事务，持有的是 行锁，还有 metadata lock 的共享锁。所以实际上，mysqldump不论你使用哪些选项，都不会阻塞事务的执行。

因为它们对锁的申请，没有任何排它性。而不像DDL一样需要持有 metadata lock 上的独占锁(排它锁)。当然DDL也会阻塞mysqldump。

mysqldump 一定需要表上的 metadata lock 共享锁。然后，要么需要所有备份表上的local读锁(lock table tb1 read local)，要么需要的是所有备份表上的全局读锁(FLUSH TABLES WITH READ LOCK;)，要么短暂持有全局锁。

