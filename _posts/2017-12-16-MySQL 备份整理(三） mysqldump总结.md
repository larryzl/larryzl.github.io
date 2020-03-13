---
layout: post
title: "MySQL 备份整理(三） mysqldump总结.md"
date: 2017-12-16 15:56:07 +0800
category: MySQL
tags: [MySQL]
---
* content
{:toc}


### 1. 利用mysqldump进行逻辑备份

1. 全逻辑备份

		mysqldump -uxxx -p --flush-logs --delete-master-logs --all-databases > alldb.sql

2. 增量备份

		mysqladmin flush-log
		
3. 缺点

	1. `--all-databases` 包含了mysql数据库，其中包含了权限的数据，所以我们应该加上`--flush-privileges`,在恢复时，权限才能生效。

		注意`--all-databases`包括了mysql数据库，但是不会包含`information_schema`和`performance_schema`两个数据库
		
	2. 因为`mysqldump`默认启用了`--lock-tables`,所以会导致在备份期间对所有表持有读锁；`lock table tb read local`,所以所有的`update`,`delete`会被阻塞。但是`select`语句喝`insert`语句不会阻塞
	3. `--delete-master-logs`备份之后，会执行`purge logs to`语句。删除了备份之后的master上的binary log。一般而言，我们不建议随便删除binary log。我们应该将他们保存起来，而不是直接删除。
	4. 该备份方式，虽然在整个备份过程中持有了`lock table tb read local`，但是还是可以执行`insert`语句的，所以得到的不是一致性的备。虽然得到的不是一致性的备份，但是因为`flush log`之后，所有的操作也会记入新的`binary log`，所以如果使用了所有新的`binary log`来金星完全恢复的话，最后恢复的数据也是一致性的。当然不一致性的备份也无法用于搭建slave。

		如果要得到一致性的备份的话，需要使用`--lock-all-tables`或者使用`--single-transaction`选项。前者使用了全局读锁，不允许修改任何操作。后者使用了时候的特许来得到一致性备份。
		
### 2. 使用mysqldump备份的最佳姿势

1. 优化锁 和 得到一致性 备份：

	我们可以结合使用 `--single-transaction` `--master-data=2` `--flush-logs` 来达到将锁定时间大大减少的目的。同时有得到了一致性的备份，而且该一致性备份和flusk的日志也是一致的；

2. 去掉`--delete-master-logs`选项，改为在备份之后，将所有被刷新的binary log移到一个地方保存起来；

3. 因为使用了`--single-transaction`选项，针对的只能是innodb数据，但是mysql数据是Myisam引擎的，所以我们最好将mysql数据库的备份分开来，另外专门针对mysql数据库进行一次操作。当然不分开来备份，可能也没有问题。
4. 还有加速 `--routines` 来备份存储过程和函数，触发器默认会备份。

优化之后，我们得到：

	mysqldump -uxxx -p --single-transaction --master-data=2 --routines --flush-logs --databases db1 db2 db3 > alldb.sql;
	
	mysqldump -uxxx -p --flush-rivileges --databases mysql > mysql.sql;

如果将mysql也一起备份的话：

	mysqldump -uxxxx -p --single-transaction --master-data=2 --routines --flush-logs --flush-rivieges --all-databases > alldb.sql
	
	
### 3. 使用mysqldummp来搭建slave环境

搭建slave环境，一般有两种方法，对于规模不大的库，可以采用mysqldump来搭建；对于规模很大的库，最好采用xtrabackup来搭建，速度要快很多。


1. 首先 分别在master 和slave 上设置不同的`server_id=1/101`,启用master上的log-bin=1,启用slave上的`relog-log=relay-bin`；在master上设置`binlog_format=row`二进制日志的格式。master上最好还设置`sync_binlog=1`和`innodb_flush_log_at-trx_commit=1`防止发生服务器崩溃时导致复制破坏。在slave上最好还配置`read-only=1`和`skip-slave-start=1`前者可以防止没有super权限的用户在slave上进行写，后者防止在启动slave数据库时，自动启动复制线程。以后需要手动start slave来启动复制线程。注意 slave没有必要启用`log-bin=1`,除非要搭建二级slave。
2. 在master上建立一个具有复制权限的用户：

		grant replication slave, replication client on *.* to repl@’192.168.%.%’ identified by ‘123456’;

3. 备份**master**上的数据库，迁移到slave上：


		mysqldump -uroot -p --routines --flush-logs --master-data=2 --databases db2 db1>/root/backup.sql
		
	因为slave的搭建需要一致性的备份，所以需要启用`--lock-all-tables`(`master-data=1/2`会自动启用`--lock-al-tables`)或者`--single-transaction`。
	
	因为`--master-data`会启用`--lock-all-tables`所以数据才是一致性的，但是导致了全局所，不能进行任何修改操作。下面我们使用`--single-transaction`进行优化：
	
		mysqldump -uroot -p --routines --flush-logs --single-transaction --master-data=2 --databases db1 db2 > /root/backup.sql; (--flush-logs非必须)

	这样全局锁仅仅在备份的开始短暂的持有，不会在备份的整个过程中持有全局锁。
	
4. 在slave上执行备份的脚本，然后连上master，开始复制线程：

	执行sql脚本：
		
		mysql> source /tmp/backup.sql
	
	找到 `--master-data`	输出的binary log的文件名和postion：
	
		# head -n 50 /tmp/backup.sql
	
		--
		-- Position to start replication or point-in-time recovery from
		--
		
		-- CHANGE MASTER TO MASTER_LOG_FILE='mysql-bin.000007', MASTER_LOG_POS=194;
		
		--

	执行`change master to `,`start slave`
	
		在slave上执行命令开始复制：
		
		mysql> change master to master_host='192.168.1.100',master_user='repl',master_password='123456',
		    -> master_log_file='mysql-bin. 000007',master_log_pos=194;
		    
		mysql> start slave;
		

	最后在slave上查看复制线程的状态：
	
		mysql> show slave status\G
		
	`slave_IO_Runing` 和 `slave_sql_runing` 状态都是yes表示搭建成功。


5. replication 涉及到的三个线程：

	1. master上的`binlog dump`（dump线程），即读取master上的binlog，发送到slave上的线程。
	2. slave上的IO线程：读取slave上的relay log。
	3. slave上的sql线程： 执行IO线程读取的relay log的线程。
	
### 4. 使用mysqldump的备份进行还原

下面使用mysqldump进行一个备份，然后删除datadir，然后使用备份sql脚本和binary log金星还原的过程。
	
1. 首先进行一个全备：

		mysqldump -uroot -p --single-transaction --master-data=2 --routines --flush-logs --databases gs ngx_lua > gs_ngx_lua_backup.sql;
		
	数据库有两个库： gs,ngx_lua

2. 将备份时刷新之后的binary log 利用mv命令移动到安全位置，也就是`--master-data=2`输出的日志文件，它之前的日志文件都存储到安全的位置。

		head -n 50 gs_ngx_lua_backup.sql
		
		--
		-- Position to start replication or point-in-time recovery from
		--
		
		-- CHANGE MASTER TO MASTER_LOG_FILE='mysql-bin.000007', MASTER_LOG_POS=194;
		
		--
		
	也就是将`MASTER_LOG_FILE='mysql-bin.000007'`之前的日志都存储到其他位置。
	然后执行： `purge binary logs to 'mysql-bin.00007'`更新了 mysql-bin.index 中的索引信息，这里并没有删除 binary log ,因为他们已经被mv走了。
	
3. 下面模拟一个 增量备份：

		mysql> delete from user where id=5;
		Query OK, 1 row affected (0.03 sec)
		
		
		mysql> select * from user;
		+----+------+
		| id | name |
		+----+------+
		|  1 | l    |
		|  2 | n    |
		|  3 | z    |
		|  4 | s    |
		|  6 | a    |
		+----+------+
		5 rows in set (0.00 sec)
		
		mysql> flush logs;
		Query OK, 0 rows affected (0.24 sec)
		
		mysql> show binary logs;
		+------------------+-----------+
		| Log_name         | File_size |
		+------------------+-----------+
		| mysql-bin.000012 |       497 |
		| mysql-bin.000013 |       194 |
		+------------------+-----------+
		2 rows in set (0.00 sec)

	这里 `flush logs`金星增量备份，然后将增量备份的`binary log`文件 `mysql-bin.000012`也存储起来。
	
	然后在进行一条delete语句：
	
		mysql> delete from user where id=4；
		Query OK, 1 row affected (0.03 sec)
		
		mysql>  select * from user;
		+----+------+
		| id | name |
		+----+------+
		|  1 | l    |
		|  2 | n    |
		|  3 | z    |
		|  6 | a    |
		+----+------+
		4 rows in set (0.00 sec)
		
	到这里数据库的最新状态是： user 表只有4条记录。

	然后我们将 `mysql-bin.000013`也存储起来。
	
4. 然后删除data-dir目录中的所有文件，开始还原

	 	
	 	# 停止mysql
	 	/etc/init.d/mysqld stop	
	 	
	 	# 备份数据
	 	rsync -av mysql/data /bak/
	 	
	 	# 删除mysql数据目录
	 	rm -rf mysql/data/*
	 	
	 为了模拟硬盘故障，也可以直接删除数据目录，kill掉mysql
	 
5. 重新初始化数据库，准备还原

		# 初始化数据库目录
		./bin/mysql_install_db --defaults-file=/etc/my.cnf --user=mysql --datadir=/Data/apps/mysql/data --basedir=/Data/apps/mysql 
		
		# 修改文件用户组
		chown -R mysql.mysql data
		
		# 初始化完成后会在根目录生成 .mysql_secret
		
		# 启动mysql
		/etc/init.d/mysqld start
		Starting MySQL.. SUCCESS! 
		
		# 登录修改mysql密码
		mysql -uroot -p
		#修改密码
		mysql>  SET PASSWORD = PASSWORD('your new password');
		Query OK, 0 rows affected, 1 warning (0.00 sec)
		# 设置过期时间
		mysql>  ALTER USER 'root'@'localhost' PASSWORD EXPIRE NEVER;
		Query OK, 0 rows affected (0.00 sec)
		# 刷新权限
		mysql>  flush privileges;
		Query OK, 0 rows affected (0.00 sec)
	
	
	开始还原数据
	
		mysql -uroot -p </bak/gs.sql 
		Enter password: 
		
	登录数据库查看数据
	
		
		mysql> use gs;
		Database changed
		mysql> select * from user;
		
	此时发现数据已经还原回来，但是他们的数据不是最新数据
	
	这是因为，我们还没有使用`mysql-bin.000012`和`mysql-bin.000013`两个binary log。 `mysql-bin.000012`是我们前面模拟的增量备份，而`mysql-bin.000013`是删除data-dir目录时，最新的binary log. 依次应用了这两个binary log之后，数据库才能恢复到最新状态。
	
6. 应用binary log

		mysqlbinlog bak/mysql-bin.000012 > 12.sql
		mysqlbinlog bak/mysql-bin.000013 > 13.sql 
		
	得到12.sql和13.sql之后，使用mysql来执行：
	
		mysql -uroot -p < 12.sql;
		
	然后查看数据
	
		mysql> select * from user;
		+----+------+
		| id | name |
		+----+------+
		|  1 | l    |
		|  2 | n    |
		|  3 | z    |
		|  4 | s    |
		|  6 | a    |
		+----+------+
		5 rows in set (0.00 sec)
	
	然后应用 13.sql得到最新状态：
	
		mysql -uroot -p <13.sql
		
	然后查看数据
	
		mysql> select * from gs.user;   
		+----+------+
		| id | name |
		+----+------+
		|  1 | l    |
		|  2 | n    |
		|  3 | z    |
		|  6 | a    |
		+----+------+
		4 rows in set (0.00 sec)
		
	可以看到，目前已经恢复到了删除之前的数据状态了


### 5. 总结

1. 逻辑备份的最佳方法：

	1. 全备：

			mysqldump -uxxx -p --single-transaction --master-data=2 --routines --flush-logs --databases db1 db2 db3 > alldb.sql;

			mysqldump -uxxx -p --flush-privileges --databases mysql > mysql.sql;
		如果将mysql也一起备份的话：
		
			mysqldump -uxxx -p --single-transaction --master-data=2 --routines --flush-logs --flush-privileges --all-databases > alldb.sql;
			
		有时，还需要加入：`--default-character-set=utf8/utf8mb4` ，该选项一般也可以配置在/etc/my.cnf中。

	2. 增量备份：flush logs; 然后将binary log存储起来即可。
		
2. 搭建slave时的最佳选项

		mysqldump -uxxx -p --single-transaction --master-data=2 --routines --databases db1 db2 db3 > alldb.sql;

搭建slave，没有必要 --flush-logs。当然搭建slave的最佳方式是使用 xtrabackup，物理备份。

3. 使用mysqldump备份的sql脚本还原的方法

先还原数据库，然后应用增量日志和最新日志，binary log在应用之前需要使用mysqlbinlog命令来处理。




	
