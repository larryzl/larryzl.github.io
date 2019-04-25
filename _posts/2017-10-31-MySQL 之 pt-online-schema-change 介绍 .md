---
layout: post
title: "MySQL 之 pt-online-schema-change 介绍"
date: 2018-10-31 10:08:16 +0800
category: MySQL 
tags: [MySQL]
---
* content
{:toc}


# 1. pt-online-schema-change 工具介绍

【安全性保障措施】

1. 默认如果检测到有复制过滤会拒绝改表，修改参数为--[no]check-replication-filters

2. 默认如果检测到主从复制延迟会自动停止数据拷贝，调节参数为--max-lag

3. 默认如果检测到服务器负载过重会停止或中断操作，调节参数为--max-load和--critical-load

4. 默认会设置锁等待超时时间为1s来避免干扰其他事务的进行，调节参数为--lock-wait-timeout

5. 默认如果检测到外键冲突后会拒绝改表，调节参数为--alter-foreign-keys-method

6. 该工具不能在PXC（Percona XtraDB Cluster）集群中对myisam表进行改表操作


**执行流程：**
>1. 如果存在外键，根据alter-foreign-keys-method参数的值，检测外键相关的表，做相应设置的处理。
2. 创建一个新的表，表结构为修改后的数据表，用于从源数据表向新表中导入数据。
3. 创建触发器，用于记录从拷贝数据开始之后，对源数据表继续进行数据修改的操作记录下来，用于数据拷贝结束后，执行这些操作，保证数据不会丢失。
4. 拷贝数据，从源数据表中拷贝数据到新表中。
5. 修改外键相关的子表，根据修改后的数据，修改外键关联的子表。
6. rename源数据表为old表，把新表rename为源表名，并将old表删除。
7. 删除触发器。

# 2. 安装
	
	yum install perl-ExtUtils-MakeMaker
	
	wget https://www.percona.com/downloads/percona-toolkit/3.0.12/source/debian/percona-toolkit-3.0.12.tar.gz
	tar -zxvf percona-toolkit-3.0.12.tar.gz
	cd percona-toolkit-3.0.12
	perl Makefile.PL
	make
	make install


# 3. 参数详解

1. --alter	
	
	用于指定改表的操作，可以同时指定多个操作，应用如下：
	
		pt-online-schema-change --alter "ADD COLUMN c1 INT" D=sakila,t=actor
		
	注意：
	
	（1）改表名时不可用RENAME（压根就不要用OSC来改表名，直接rename多好）

	（2）修改列名时不可以通过drop该列然后重新add该列来完成，这样会导致无法拷贝原始列的数据出来
	
	（3）如果add一个列没设默认值且设置为‘not null’，工具会拒绝执行，因为工具不会帮你猜一个默认值出来
	
	（4）drop外键时，外键名字前需要加 ‘_’ 而不是单纯的就一个外键名字，如drop下面这个外键时
		
		CONSTRAINT `fk_foo` FOREIGN KEY (`foo_id`) REFERENCES `bar` (`foo_id`)
		
		#必须这样指定：
		
		--alter "DROP FOREIGN KEY _fk_foo"

	（5） MySQL版本为5.0时需要注意，该工具不能使用LOCK IN SHARE MODE，这会引起一个复制错误，报错如下：
		
		Query caused different errors on master and slave. Error on master:
		'Deadlock found when trying to get lock; try restarting transaction' (1213),
		Error on slave: 'no error' (0). Default database: 'pt_osc'.
		Query: 'INSERT INTO pt_osc.t (id, c) VALUES ('730', 'new row')'

		
		一般发生在转换myisam表为innodb表时，因为myisam表不支持事务而innodb表支持。5.1及更新版本没有该问题

2. --alter-foreign-keys-method

	外键改表前后必须持续的链接正确的表，当该工具rename原始表并用新表来取代原始表时，外键必须正确更新到新表上，并且原始表中的外键不再生效

	**有两种方法来实现这个目的，具体参数有四：**

	（1）auto
	
	自动决定采用哪个方法，如果可以就采用rebuild_constraints，如果不可以就采用drop_swap
	
	（2）rebuild_constraints
	
	 该方法采用alter table来drop并re-add链接新表的外键。除非相关的子表太大使得alter过程花费时间过长，一般都采用该方法。这里的花费时间是通过比较子表中的行数和该工具将原始表数据拷贝到新表中的拷贝速率来评估的，如果评估后发现子表中数据能够在少于--chunk-time的时间内alter完成，就会采用该方法。另外，因为在MySQL中alter table比外部拷贝数据的速率快很多，所以拷贝速率是按照--chunk-size-limit来决定的
	
	因为MySQL的限制，外键在改表前后的名字会不一样，改表后新表中的外键名前会加一个下划线，同样，会自动的更改外键相应的索引名字
	
	（3）drop_swap
	
	该方法禁止外键检查(FOREIGN_KEY_CHECKS=0)，然后在rename新表之前就将原始表drop掉，这个方法更快而且不会被阻塞，但是风险比较大，风险有二：
	
	- 在drop掉原始表和rename新表之间有一个时间差，在这段时间里这个表是不存在的，这会导致查询报错
	
	- 如果rename新表时发生了错误，那问题就大了，因为原始表已经被drop掉了，只能呵呵了
	
	（4）none
	
	这个方法类似没有“swap”的drop_swap，原始表中的所有外键都会被指定到一个不存在的表上，这样会导致查询show engine innodb status时会显示如下信息：
	
		Trying to add to index `idx_fk_staff_id` tuple:
		DATA TUPLE: 2 fields;
		0: len 1; hex 05; asc  ;;
		1: len 4; hex 80000001; asc     ;;
		But the parent table `sakila`.`staff_old`
		or its .ibd file does not currently exist!

	因为原始表(database.tablename)会被rename为database.tablename_old然后drop掉。这种处理外键的方法可以让DBA在需要时取消该工具的这种内置功能

3. --charset

	默认字符类型，例如如果值为utf8，就将输出的字符设置为utf8格式，将mysql_enable_utf8传递给DBD::mysql，然后连接MySQL后运行SET NAMES UTF8命令

 

4.  --[no]check-alter

	用于改表安全警告，目前支持两种报警：
	
	（1）Column renames
	
	该工具以前的版本中，用CHANGE COLUMN name new_name来改变列名时会导致该列的数据丢失，后来的版本中修补了这个bug，但是这段代码仍然不够万无一失，所以在操作前最好用--dry-run 和 --print去检测一下确定可以正确的rename column
	
	（2）DROP PRIMARY KEY
	
	除非指定--dry-run，否则一旦执行DROP PRIMARY KEY就会发出警告。该工具的触发器，尤其是delete trigger，主要是l利用主键来执行触发的，所以会受到很大的影响。所以最好先用--dry-run 和 --print去检测一下确保trigger可以正确执行

 

5. --[no]check-replication-filters

	如果主从复制中有复制过滤就报错退出，即如果检测到有binlog_ignore_db、replicate_do_db等过滤手段，就报错退出

 

6. --check-slave-lag

	如果复制延迟大于--max-lag就停止数据拷贝，设置该参数后会监控所有的复制进程，一旦发现任何大于--max-lag时间的就停止数据拷贝，如果不想监控所有的复制进程，可以通过--recursion-method来指定想要监控的对象
	
7. --recursion-method

	master寻找slave的方法，默认值为processlist,hosts，所有方法如下：
			
		METHOD       USES
		===========  ==================
		processlist  SHOW PROCESSLIST
		hosts        SHOW SLAVE HOSTS
		dsn=DSN      DSNs from a table
		none         Do not find slaves
		
	（1）processlist是默认的，因为show slave status不太靠谱。
	
	（2）如果服务器使用非3306端口时，hosts方法也很好
	
	（3）dsn=DSN方法使用时，需要先去库里创建一个表，比如在percona库中建一个dnsn表

	建表语句是： 
	
		CREATE TABLE `dsns` (`id` int(11) NOT NULL AUTO_INCREMENT,`parent_id` int(11) DEFAULT NULL,`dsn` varchar(255) NOT NULL,PRIMARY KEY (`id`));
		
	建好后插入主从复制信息数据，如：insert into table dsns(dsn) values(h=slave_host,u=repl_user,p=repl_password,P=port );

    然后就可以使用DSN方法了：命令为：--recursion-method dsn=D=percona,t=dsns.

	（4）如果想只监控hosts10.10.1.16 and 10.10.1.17的复制延迟情况，可以将h=10.10.1.16 and h=10.10.1.17的相关内容存入‘dsns’的表中来精确指定

8. --chunk-time

	默认是0.5秒，工具会根据当前系统运行繁忙程度计算出在该指定时间内可以处理的数据行数（即chunk），比较灵活

9. --dry-run

	create并alter新表，但不创建trigger，不拷贝数据以及替换原始表

10. --execute

	该参数用于执行alter操作，如果不加的话，只会做一些安全检查后退出。一定要确保知道如何使用该工具并有合适的备份后，再添加该参数。使用该参数时，除了对对象表所需的权限外，还需要SUPER, REPLICATION SLAVE两种权限。

11. --max-lag（type: time; default: 1s）

	中断数据拷贝直到所有的复制延迟都少于这个值，默认为1S。每一个chunk拷贝完成后，OSC都会去show salve 	status通过Seconds_Behind_Master来确定所有的复制情况，任何相关的slave的复制延迟高于该值时，OSC就会停止数据拷贝--check-interval参数所指定的时间，然后重新发起检查，直到延迟降低到该值以下。

	如果指定了--check-slave-lag参数，OSC则只检查指定的slave延迟情况，而不是所有的slave。如果想精确的指定监控服务器，可以用--recursion-method的DSN去指定

12. --max-load(type: Array; default: Threads_running=25)

	该参数的目的是为了防止OSC执行过程造成的负载太大，如果数据拷贝引起了锁等待或者负载过大，其他查询会被阻塞并形成队列，这会引起Threads_running增长。OSC会在每次chunk拷贝完成后执行show global status来检测系统状态，如果设置了一些系统参数的临界值，一旦检测到超过临界值，osc就停止数据拷贝直到检测值回到临界值以下。如果这样还是发现有阻塞，那最好把chunk time值调小

	在每个chunk拷贝后都执行show global status进行检查，如果发现状态参数高于它们的临界值就中断拷贝，该参数接受一个以逗号分隔的MySQL状态参数list，参数赋值方式为variable_name=max_value或variable_name:max_value，如果不指定，OSC会将参数值定为当前参数值的120%

	即：如果指定Threads_connected参数，不赋值且当前该参数的值是100，则Threads_connected超过120时就会中断数据拷贝，如果想精确指定的话，就用‘=’或‘：’去精确赋值

 

【其他参数】

1. --ask-pass            

	Prompt for a password when connecting to MySQL

2. --check-interval（type: time; default: 1）

	Sleep time between checks for --max-lag.

3. --chunk-index

	用于指定索引

4. --chunk-index-columns

	用于指定联合索引的采用列

5. --chunk-size

	用于指定chunk的大小，不推荐使用，建议用--chunk-time

6. --chunk-size-limit

	用于限制chunk的大小

7. --config

	指定配置，如果要用该参数，必须放在命令行的最前面

8. --critical-load

	类似于--max-load，不同的是检测到超高负载时会直接中断OSC进程而不是暂停

9. --default-engine

	指定引擎，采用才参数后会使新表采用MySQL当前的默认存储引擎而不是原始表的

10. --defaults-file

	指定配置文件，必须用绝对路径

11. --[no]drop-new-table

	默认在拷贝失败后会将新表drop掉，可以用该参数配置

12. --[no]drop-old-table

	默认在改表结束后会将原始表drop掉，可以用该参数配置

13. --host，--password，--pid，--port，--socket，--user

	连接MySQL数据库的各参数

14. --print

	将改表过程中的SQL语句输出到STDOUT，观察改表过程中的操作时用这个参数配合--dry-run是极好的

15. --progress

	在拷贝数据过程中将进程报告输出到STDERR，参数的指定由两部分组成，第一部分是percentage, time, or iterations，第二部分是分别对应钱第一部分的数值指定，即百分值，秒数和次数，默认是time,30

16. --quiet

	不要输出信息到STDOUT，错误和警告信息仍会输出到STDERR

17. --recurse

	复制层级的指定，默认是无限层级

18. --retries

	当数据拷贝过程中有非致命性错误，如锁等待超时或查询被kill等情况时，重新尝试的次数。默认是3

19. --set-vars

	用set ........直接将该指定参数中的值在MySQL客户端中执行

20. --[no]swap-tables

	默认在改表结束后用新表来替换原始表，可以用该参数配置



# 4. 常用操作

#### 4.1. 添加字段

	HOST=xx.x.x.x
	USER=xxx
	PORT=3306
	PWD=xxx
	DBName=
	Table=
	SQL="add column fld_ggz_uid int(11) unsigned not null default 0"
	
	pt-online-schema-change --host=${HOST} --port=${PORT} \
		--user=${USER} --password=${PWD} --alter=${SQL} \
		--execute D=${DBName},t=${Table} --set-vars innodb_lock_wait_timeout=50 \
		--no-check-replication-filters --charset=utf8mb4

#### 4.2. 修改字段

	HOST=xx.x.x.x
	USER=xxx
	PORT=3306
	PWD=xxx
	DBName=
	Table=
	SQL="modify column C_N varchar(128) default 0"
	
	pt-online-schema-change --host=${HOST} --port=${PORT} \
		--user=${USER} --password=${PWD} --alter=${SQL} \
		--execute D=${DBName},t=${Table} --set-vars innodb_lock_wait_timeout=50 \
		--no-check-replication-filters --charset=utf8mb4

#### 4.3. 改字段名

	HOST=xx.x.x.x
	USER=xxx
	PORT=3306
	PWD=xxx
	DBName=
	Table=
	SQL="hange C_N C_N_01 varchar(128) default 11"
	
	pt-online-schema-change --host=${HOST} --port=${PORT} \
		--user=${USER} --password=${PWD} --alter=${SQL} \
		--execute D=${DBName},t=${Table} --set-vars innodb_lock_wait_timeout=50 \
		--no-check-replication-filters --charset=utf8mb4

#### 4.4. 删除字段

	HOST=xx.x.x.x
	USER=xxx
	PORT=3306
	PWD=xxx
	DBName=
	Table=
	SQL="drop column C_N_01"
	
	pt-online-schema-change --host=${HOST} --port=${PORT} \
		--user=${USER} --password=${PWD} --alter=${SQL} \
		--execute D=${DBName},t=${Table} --set-vars innodb_lock_wait_timeout=50 \
		--no-check-replication-filters --charset=utf8mb4

#### 4.5. 添加索引

	HOST=xx.x.x.x
	USER=xxx
	PORT=3306
	PWD=xxx
	DBName=
	Table=
	SQL="add index IDX_MOBILE(MOBILE)"
	
	pt-online-schema-change --host=${HOST} --port=${PORT} \
		--user=${USER} --password=${PWD} --alter=${SQL} \
		--execute D=${DBName},t=${Table} --set-vars innodb_lock_wait_timeout=50 \
		--no-check-replication-filters --charset=utf8mb4

#### 4.6. 删除索引

	HOST=xx.x.x.x
	USER=xxx
	PORT=3306
	PWD=xxx
	DBName=
	Table=
	SQL="drop index IDX_MOBILE"
	
	pt-online-schema-change --host=${HOST} --port=${PORT} \
		--user=${USER} --password=${PWD} --alter=${SQL} \
		--execute D=${DBName},t=${Table} --set-vars innodb_lock_wait_timeout=50 \
		--no-check-replication-filters --charset=utf8mb4

# 5. 总结

在没有执行sql语句时，mysql在线修改表结构的时间与pt-online-schma-change的基本上相等，无大的差别；pt-online-schma-change方式会占用较多内存，负载也会略高   
在mysql预热以后，在线修改表结构的时间，直接修改会比pt-online-schema-change方式略快 
在线修改表结构的同时执行mysql语句，mysql直接修改的方式会先修改完表结构再执行sql语句，pt-online-schema-change方式会优先执行sql语句，再复制数据表，复制完毕后再把执行sql语句的结果更新到新表，因此，在时间上，直接修改表结构会比pt-online-schema-change方式，在实际用时上，直接修改会略快。

**这个过程中有两个问题需要注意：**

1. 触发器
	
	因为整个过程是在线的，为了将改表过程中对原始表的更新同时更新到新表上，会创建相应的触发器，每当发生针对原始表的增删改操作，就会触发对新表的相应的操作。所以原始表上不能有其他触发器，即如果原始表上存有触发器，OSC会罢工的

2. 外键

	外键使改表操作变得更加复杂，如果原始表上有外键的话，自动rename原始表和新表的操作就不能顺利进行，必须要在数据拷贝完成后将外键更新到新表上，该工具有两种方法来支持这个操作，具体后面参数部分（--alter-foreign-keys-method）介绍
