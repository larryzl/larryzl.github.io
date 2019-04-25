---
layout: post
title: "MySQL 备份整理(六）xtrabackup使用(转）"
date: 2017-12-25 16:35:51 +0800
category: MySQL
tags: [MySQL]
---
* content
{:toc}


>背景：
>
>  关于物理备份工具xtrabackup的一些说明可以先看之前写过的文章说明：[xtrabackup 安装使用](http://www.cnblogs.com/zhoujinyi/p/4088866.html)。现在xtrabackup版本升级到了2.4.4，相比之前的2.1有了比较大的变化：innobackupex 功能全部集成到 xtrabackup 里面，只有一个 binary，另外为了使用上的兼容考虑，innobackupex作为 xtrabackup 的一个软链，即xtrabackup现在支持非Innodb表备份，并且Innobackupex在下一版本中移除，建议通过xtrabackup替换innobackupex。还有其他的一些新特性，更多的说明可以看[xtrabackup新版详细说明](http://www.cnblogs.com/billyxp/p/5305676.html)，原理说明可以看[Percona XtraBackup 备份原理说明【转](http://www.cnblogs.com/zhoujinyi/p/5888271.html)】，本篇文章将介绍xtrabackup的使用方法。


### innobackupex参数说明（innobackupex --help ）：

#### 1. 备份

	innobackupex [--compress] [--compress-threads=NUMBER-OF-THREADS] [--compress-chunk-size=CHUNK-SIZE]
	             [--encrypt=ENCRYPTION-ALGORITHM] [--encrypt-threads=NUMBER-OF-THREADS] [--encrypt-chunk-size=CHUNK-SIZE]
	             [--encrypt-key=LITERAL-ENCRYPTION-KEY] | [--encryption-key-file=MY.KEY]
	             [--include=REGEXP] [--user=NAME]
	             [--password=WORD] [--port=PORT] [--socket=SOCKET]
	             [--no-timestamp] [--ibbackup=IBBACKUP-BINARY]
	             [--slave-info] [--galera-info] [--stream=tar|xbstream]
	             [--defaults-file=MY.CNF] [--defaults-group=GROUP-NAME]
	             [--databases=LIST] [--no-lock] 
	             [--tmpdir=DIRECTORY] [--tables-file=FILE]
	             [--history=NAME]
	             [--incremental] [--incremental-basedir]
	             [--incremental-dir] [--incremental-force-scan] [--incremental-lsn]
	             [--incremental-history-name=NAME] [--incremental-history-uuid=UUID]
	             [--close-files] [--compact]     
	             BACKUP-ROOT-DIR

##### 参数说明

	--compress：该选项表示压缩innodb数据文件的备份。
	--compress-threads：该选项表示并行压缩worker线程的数量。
	--compress-chunk-size：该选项表示每个压缩线程worker buffer的大小，单位是字节，默认是64K。
	--encrypt：该选项表示通过ENCRYPTION_ALGORITHM的算法加密innodb数据文件的备份，目前支持的算法有ASE128,AES192,AES256。
	--encrypt-threads：该选项表示并行加密的worker线程数量。
	--encrypt-chunk-size：该选项表示每个加密线程worker buffer的大小，单位是字节，默认是64K。
	--encrypt-key：该选项使用合适长度加密key，因为会记录到命令行，所以不推荐使用。
	--encryption-key-file：该选项表示文件必须是一个简单二进制或者文本文件，加密key可通过以下命令行命令生成：openssl rand -base64 24。
	--include：该选项表示使用正则表达式匹配表的名字[db.tb]，要求为其指定匹配要备份的表的完整名称，即databasename.tablename。
	--user：该选项表示备份账号。
	--password：该选项表示备份的密码。
	--port：该选项表示备份数据库的端口。
	--host：该选项表示备份数据库的地址。
	--databases：该选项接受的参数为数据名，如果要指定多个数据库，彼此间需要以空格隔开；如："xtra_test dba_test"，同时，在指定某数据库时，也可以只指定其中的某张表。如："mydatabase.mytable"。该选项对innodb引擎表无效，还是会备份所有innodb表。此外，此选项也可以接受一个文件为参数，文件中每一行为一个要备份的对象。
	--tables-file：该选项表示指定含有表列表的文件，格式为database.table，该选项直接传给--tables-file。
	--socket：该选项表示mysql.sock所在位置，以便备份进程登录mysql。
	--no-timestamp：该选项可以表示不要创建一个时间戳目录来存储备份，指定到自己想要的备份文件夹。
	--ibbackup：该选项指定了使用哪个xtrabackup二进制程序。IBBACKUP-BINARY是运行percona xtrabackup的命令。这个选项适用于xtrbackup二进制不在你是搜索和工作目录，如果指定了该选项,innoabackupex自动决定用的二进制程序。
	--slave-info：该选项表示对slave进行备份的时候使用，打印出master的名字和binlog pos，同样将这些信息以change master的命令写入xtrabackup_slave_info文件。可以通过基于这份备份启动一个从库。
	--safe-slave-backup：该选项表示为保证一致性复制状态，这个选项停止SQL线程并且等到show status中的slave_open_temp_tables为0的时候开始备份，如果没有打开临时表，bakcup会立刻开始，否则SQL线程启动或者关闭知道没有打开的临时表。如果slave_open_temp_tables在--safe-slave-backup-timeount（默认300秒）秒之后不为0，从库sql线程会在备份完成的时候重启。
	--rsync：该选项表示通过rsync工具优化本地传输，当指定这个选项，innobackupex使用rsync拷贝非Innodb文件而替换cp，当有很多DB和表的时候会快很多，不能--stream一起使用。
	--kill-long-queries-timeout：该选项表示从开始执行FLUSH TABLES WITH READ LOCK到kill掉阻塞它的这些查询之间等待的秒数。默认值为0，不会kill任何查询，使用这个选项xtrabackup需要有Process和super权限。
	--kill-long-query-type：该选项表示kill的类型，默认是all，可选select。
	--ftwrl-wait-threshold：该选项表示检测到长查询，单位是秒，表示长查询的阈值。
	--ftwrl-wait-query-type：该选项表示获得全局锁之前允许那种查询完成，默认是ALL，可选update。
	--galera-info：该选项表示生成了包含创建备份时候本地节点状态的文件xtrabackup_galera_info文件，该选项只适用于备份PXC。
	--stream：该选项表示流式备份的格式，backup完成之后以指定格式到STDOUT，目前只支持tar和xbstream。
	--defaults-file：该选项指定了从哪个文件读取MySQL配置，必须放在命令行第一个选项的位置。
	--defaults-extra-file：该选项指定了在标准defaults-file之前从哪个额外的文件读取MySQL配置，必须在命令行的第一个选项的位置。一般用于存备份用户的用户名和密码的配置文件。
	----defaults-group：该选项表示从配置文件读取的组，innobakcupex多个实例部署时使用。
	--no-lock：该选项表示关闭FTWRL的表锁，只有在所有表都是Innodb表并且不关心backup的binlog pos点，如果有任何DDL语句正在执行或者非InnoDB正在更新时（包括mysql库下的表），都不应该使用这个选项，后果是导致备份数据不一致，如果考虑备份因为获得锁失败，可以考虑--safe-slave-backup立刻停止复制线程。
	--tmpdir：该选项表示指定--stream的时候，指定临时文件存在哪里，在streaming和拷贝到远程server之前，事务日志首先存在临时文件里。在 使用参数stream=tar备份的时候，你的xtrabackup_logfile可能会临时放在/tmp目录下，如果你备份的时候并发写入较大的话 xtrabackup_logfile可能会很大(5G+)，很可能会撑满你的/tmp目录，可以通过参数--tmpdir指定目录来解决这个问题。
	--history：该选项表示percona server 的备份历史记录在percona_schema.xtrabackup_history表。
	--incremental：该选项表示创建一个增量备份，需要指定--incremental-basedir。
	--incremental-basedir：该选项表示接受了一个字符串参数指定含有full backup的目录为增量备份的base目录，与--incremental同时使用。
	--incremental-dir：该选项表示增量备份的目录。
	--incremental-force-scan：该选项表示创建一份增量备份时，强制扫描所有增量备份中的数据页。
	--incremental-lsn：该选项表示指定增量备份的LSN，与--incremental选项一起使用。
	--incremental-history-name：该选项表示存储在PERCONA_SCHEMA.xtrabackup_history基于增量备份的历史记录的名字。Percona Xtrabackup搜索历史表查找最近（innodb_to_lsn）成功备份并且将to_lsn值作为增量备份启动出事lsn.与innobackupex--incremental-history-uuid互斥。如果没有检测到有效的lsn，xtrabackup会返回error。
	--incremental-history-uuid：该选项表示存储在percona_schema.xtrabackup_history基于增量备份的特定历史记录的UUID。
	--close-files：该选项表示关闭不再访问的文件句柄，当xtrabackup打开表空间通常并不关闭文件句柄目的是正确的处理DDL操作。如果表空间数量巨大，这是一种可以关闭不再访问的文件句柄的方法。使用该选项有风险，会有产生不一致备份的可能。
	--compact：该选项表示创建一份没有辅助索引的紧凑的备份。
	--throttle：该选项表示每秒IO操作的次数，只作用于bakcup阶段有效。apply-log和--copy-back不生效不要一起用。

#### 2. prepare

	innobackupex --apply-log [--use-memory=B]
	             [--defaults-file=MY.CNF]
	             [--export] [--redo-only] [--ibbackup=IBBACKUP-BINARY]
	             BACKUP-DIR

##### 参数说明：

	--apply-log：该选项表示同xtrabackup的--prepare参数,一般情况下,在备份完成后，数据尚且不能用于恢复操作，因为备份的数据中可能会包含尚未提交的事务或已经提交但尚未同步至数据文件中的事务。因此，此时数据 文件仍处理不一致状态。--apply-log的作用是通过回滚未提交的事务及同步已经提交的事务至数据文件使数据文件处于一致性状态。
	--use-memory：该选项表示和--apply-log选项一起使用，prepare 备份的时候，xtrabackup做crash recovery分配的内存大小，单位字节。也可(1MB,1M,1G,1GB)，推荐1G。
	--defaults-file：该选项指定了从哪个文件读取MySQL配置，必须放在命令行第一个选项的位置。
	--export：这个选项表示开启可导出单独的表之后再导入其他Mysql中。
	--redo-only：这个选项在prepare base full backup，往其中merge增量备份（但不包括最后一个）时候使用。

#### 3. 解压解密：

	innobackupex [--decompress] [--decrypt=ENCRYPTION-ALGORITHM]
	             [--encrypt-key=LITERAL-ENCRYPTION-KEY] | [--encryption-key-file=MY.KEY]
	             [--parallel=NUMBER-OF-FORKS] BACKUP-DIR

##### 参数说明：

	--decompress：该选项表示解压--compress选项压缩的文件。
	--parallel：该选项表示允许多个文件同时解压。为了解压，qpress工具必须有安装并且访问这个文件的权限。这个进程将在同一个位置移除原来的压缩/加密文件。
	--decrypt：该选项表示解密通过--encrypt选项加密的.xbcrypt文件。

#### 4. 还原

	innobackupex --copy-back [--defaults-file=MY.CNF] [--defaults-group=GROUP-NAME] BACKUP-DIR
	innobackupex --move-back [--defaults-file=MY.CNF] [--defaults-group=GROUP-NAME] BACKUP-DIR

##### 参数说明：

	--copy-back：做数据恢复时将备份数据文件拷贝到MySQL服务器的datadir。
	--move-back：这个选项与--copy-back相似，唯一的区别是它不拷贝文件，而是移动文件到目的地。这个选项移除backup文件，用时候必须小心。使用场景：没有足够的磁盘空间同事保留数据文件和Backup副本
	注意：
	1.datadir目录必须为空。除非指定innobackupex --force-non-empty-directorires选项指定，否则--copy-backup选项不会覆盖
	2.在restore之前,必须shutdown MySQL实例，你不能将一个运行中的实例restore到datadir目录中
	3.由于文件属性会被保留，大部分情况下你需要在启动实例之前将文件的属主改为mysql，这些文件将属于创建备份的用户
	chown -R my5711:mysql /data1/dbrestore
	以上需要在用户调用Innobackupex之前完成
	
	--force-non-empty-directories：指定该参数时候，使得innobackupex --copy-back或--move-back选项转移文件到非空目录，已存在的文件不会被覆盖。如果--copy-back和--move-back文件需要从备份目录拷贝一个在datadir已经存在的文件，会报错失败。

#### 5. 应用场景：

1. 普通全量备份、还原：（库：dba_test、xtra_test）

		#备份所有数据库：备份目录里生成日期命名的文件夹
		innobackupex --defaults-file=/etc/mysql/my.cnf --user=root --password=123 /home/zhoujy/xtrabackup/
		#还原
		1.先prepare，利用--apply-log的作用是通过回滚未提交的事务及同步已经提交的事务至数据文件使数据文件处于一致性状态
		innobackupex --apply-log /home/zhoujy/xtrabackup/2016-09-23_10-53-51/
		2.copy：需要数据目录为空
		innobackupex --defaults-file=/etc/mysql/my.cnf --copy-back /home/zhoujy/xtrabackup/2016-09-23_10-53-51/
		3.改权限
		++++++++++++++++++
		#备份所有数据库：指定备份目录
		innobackupex --defaults-file=/etc/mysql/my.cnf --user=root --password=123 --no-timestamp /home/zhoujy/xtrabackup/
		#还原同上
		
		++++++++++++++++++
		#备份指定数据库名，多个数据库用空格分开
		innobackupex --defaults-file=/etc/mysql/my.cnf --user=root --password=123 --no-timestamp --databases="dba_test xtra_test" /home/zhoujy/xtrabackup/
		#还原
		1.先prepare，利用--apply-log的作用是通过回滚未提交的事务及同步已经提交的事务至数据文件使数据文件处于一致性状态
		innobackupex --apply-log /home/zhoujy/xtrabackup/
		2.copy，因为是部分备份，不能直接用--copy-back，只能手动来复制需要的库，也要复制ibdata(数据字典）
		cp -r dba_test/ /var/lib/mysql/
		cp -r xtrabackup/dba_test/ /var/lib/mysql
		3.改权限
		++++++++++++++++++
		#备份指定表
		备份不同库下的不同表
		1：innobackupex --defaults-file=/etc/mysql/my.cnf --user=root --password=123 --no-timestamp --databases="dba_test.tb1 xtra_test.M" /home/zhoujy/xtrabackup/
		备份一个库下面的表，支持正则，如：--include='^mydatabase[.]mytable' 
		2：innobackupex --defaults-file=/etc/mysql/my.cnf --user=root --password=123 --no-timestamp --include='xtra_test.I' /home/zhoujy/xtrabackup/
		备份指定文件里的表，文件里每行的格式是：dbname.tbname
		3：innobackupex --defaults-file=/etc/mysql/my.cnf --user=root --password=123 --no-timestamp --tables-file=/tmp/tbname.txt  /home/zhoujy/xtrabackup/
		#还原
		同指定数据库还原一样，需要还原ibdata。注意，还原的时候可以这样还原：
		innobackupex --apply-log --export xtrabackup/
		生成如下几个文件：
		-rw-r--r-- 1 root root  425  9月 23 17:16 I.cfg
		-rw-r----- 1 root root  16K  9月 23 17:16 I.exp
		-rw-r----- 1 root root 8.4K  9月 23 17:15 I.frm
		-rw-r----- 1 root root  96K  9月 23 17:15 I.ibd
		
		  然后：
		  alter table I discard tablespace;
		  将I.exp和I文件传到目标机目标目录中执行:
		  alter table I import tablespace;
	

2. 普通增量备份、还原

		#全量备份，这里举例单个表，也可以是指定几个库，甚至所有库
		innobackupex --defaults-file=/etc/mysql/my.cnf --user=root --password=123 --no-timestamp --databases="xtra_test.I" /home/zhoujy/xtrabackup/
		#增量备份1
		innobackupex --defaults-file=/etc/mysql/my.cnf --user=root --password=123 --no-timestamp --databases="xtra_test.I" --incremental-basedir=/home/zhoujy/xtrabackup/  --incremental /home/zhoujy/increment_data/
		#增量备份2
		innobackupex --defaults-file=/etc/mysql/my.cnf --user=root --password=123 --no-timestamp --databases="xtra_test.I" --incremental-basedir=/home/zhoujy/increment_data/ --incremental /home/zhoujy/increment_data1/
		
		####信息####
		通过上面三个目录里的xtrabackup_checkpoints文件，可以看出是哪种备份类型，全量（full-backuped）还是增量（incremental）。并且全量到增量的from_lsn和last_lsn是一一对应的。
		在第2次做增量备份的时候 --incremental-basedir 指向全量备份，则第一次增量备份中的数据会被第2次包含，只需要还原一次就可以恢复，现在则需要还原2次增量备份。
		##########
		
		#还原
		1.先prepare全备
		innobackupex --incremental --apply-log --redo-only /home/zhoujy/xtrabackup
		2.再prepare第一个增量
		innobackupex --incremental --apply-log --redo-only /home/zhoujy/xtrabackup/ --user-memory=1G  --incremental-dir=/home/zhoujy/increment_data/
		3.然后prepare最后一个增量
		innobackupex --incremental --apply-log --redo-only /home/zhoujy/xtrabackup/ --user-memory=1G  --incremental-dir=/home/zhoujy/increment_data1/
		
		通过上面额可以看到全量备份里xtrabackup_checkpoints文件的to_lsn是最新的lsn。
		
		4.最后再prepare全量备份
		innobackupex --apply-log /home/zhoujy/xtrabackup/
		
		5.copy
		因为是部分备份，不是所有库备份，所以和上面介绍的一样，先手动复制需要的文件再修改权限即可恢复数据。

3. 打包压缩备份，注意：--compress不能和--stream=tar一起使用

		#压缩备份
		innobackupex --defaults-file=/etc/mysql/my.cnf --user=root --password=123 --compress --compress-threads=8 --no-timestamp --databases="xtra_test.I" /home/zhoujy/xtrabackup/
		
		#在perpare之前需要decompress，需要安装qpress
		innobackupex --decompress /home/zhoujy/xtrabackup/
		
		#prepare
		innobackupex --apply-log /home/zhoujy/xtrabackup/
		
		最后还原方法和上面一致
		
		#打包备份
		innobackupex --defaults-file=/etc/mysql/my.cnf --user=root --password=123 --stream=tar --no-timestamp --databases="xtra_test" /home/zhoujy/xtrabackup/ 1>/home/zhoujy/xtrabackup/xtra_test.tar
		
		#解包
		tar ixvf xtra_test.tar
		
		最后还原方法和上面一致
		
		#第三方压缩备份：
		innobackupex --defaults-file=/etc/mysql/my.cnf --user=root --password=123 --compress --compress-threads=8 --parallel=4 --stream=tar --no-timestamp --databases="xtra_test" /home/zhoujy/xtrabackup/ | gzip >/home/zhoujy/xtrabackup/xtra_test.tar.gz
		
		#prepare之前先解压
		tar izxvf xtra_test.tar.gz 
		
		#prepare
		innobackupex --apply-log /home/zhoujy/xtrabackup/

4. 加密备份

		说明：在参数说明里看到加密备份的几个参数：--encrypt、--encrypt-threads、--encrypt-key、--encryption-key-file。其中encrypt-key和encryption-key-file不能一起使用，encryption-key需要把加密的密码写到命令行，不推荐。
		
		#加密备份：
		先生成key:
		openssl rand -base64 24
		把Key写到文件：
		echo -n "Ue2Wp6dIDWszpI76HQ1u57exyjAdHpRO" > keyfile 
		最后备份：
		innobackupex --defaults-file=/etc/mysql/my.cnf --user=root --password=123 --compress --compress-threads=3 --no-timestamp --encrypt=AES256 --encrypt-key-file=/home/zhoujy/keyfile ----encrypt-threads=3 --parallel=5 /home/zhoujy/xtrabackup2/
		
		#解密：
		for i in `find . -iname "*\.xbcrypt"`; do xbcrypt -d --encrypt-key-file=/home/zhoujy/keyfile --encrypt-algo=AES256 < $i > $(dirname $i)/$(basename $i .xbcrypt) && rm $i; done
		
		#解压：
		innobackupex --decompress /home/zhoujy/xtrabackup2/
		
		#prepare：
		innobackupex --apply-log /home/zhoujy/xtrabackup2/
		
		#还原copy
		innobackupex --defaults-file=/etc/mysql/my.cnf --move-back /home/zhoujy/xtrabackup2/

5. 复制环境中的备份：一般生产环境大部分都是主从模式，主提供服务，从提供备份。

		#备份 5个线程备份2个数据库，并且文件xtrabackup_slave_info记录GTID和change的信息
		innobackupex --defaults-file=/etc/mysql/my.cnf --user=root --password=123 --no-timestamp --slave-info --safe-slave-backup --parallel=5 --databases='xtra_test dba_test' /home/zhoujy/xtrabackup/
		
		#还原
		还原方法同上

### xtrabackup 参数说明（xtrabackup --help ）： 
	
	--apply-log-only：prepare备份的时候只执行redo阶段，用于增量备份。
	--backup：创建备份并且放入--target-dir目录中
	--close-files：不保持文件打开状态，xtrabackup打开表空间的时候通常不会关闭文件句柄，目的是为了正确处理DDL操作。如果表空间数量非常巨大并且不适合任何限制，一旦文件不在被访问的时候这个选项可以关闭文件句柄.打开这个选项会产生不一致的备份。
	--compact：创建一份没有辅助索引的紧凑备份
	--compress：压缩所有输出数据，包括事务日志文件和元数据文件，通过指定的压缩算法，目前唯一支持的算法是quicklz.结果文件是qpress归档格式，每个xtrabackup创建的*.qp文件都可以通过qpress程序提取或者解压缩
	--compress-chunk-size=#：压缩线程工作buffer的字节大小，默认是64K
	--compress-threads=#：xtrabackup进行并行数据压缩时的worker线程的数量，该选项默认值是1，并行压缩（'compress-threads'）可以和并行文件拷贝('parallel')一起使用。例如:'--parallel=4 --compress --compress-threads=2'会创建4个IO线程读取数据并通过管道传送给2个压缩线程。
	--create-ib-logfile：这个选项目前还没有实现，目前创建Innodb事务日志，你还是需要prepare两次。
	--datadir=DIRECTORY：backup的源目录，mysql实例的数据目录。从my.cnf中读取，或者命令行指定。
	--defaults-extra-file=[MY.CNF]：在global files文件之后读取，必须在命令行的第一选项位置指定。
	--defaults-file=[MY.CNF]：唯一从给定文件读取默认选项，必须是个真实文件，必须在命令行第一个选项位置指定。
	--defaults-group=GROUP-NAME：从配置文件读取的组，innobakcupex多个实例部署时使用。
	--export：为导出的表创建必要的文件
	--extra-lsndir=DIRECTORY：(for --bakcup):在指定目录创建一份xtrabakcup_checkpoints文件的额外的备份。
	--incremental-basedir=DIRECTORY：创建一份增量备份时，这个目录是增量别分的一份包含了full bakcup的Base数据集。
	--incremental-dir=DIRECTORY：prepare增量备份的时候，增量备份在DIRECTORY结合full backup创建出一份新的full backup。
	--incremental-force-scan：创建一份增量备份时，强制扫描所有增在备份中的数据页即使完全改变的page bitmap数据可用。
	--incremetal-lsn=LSN：创建增量备份的时候指定lsn。
	--innodb-log-arch-dir：指定包含归档日志的目录。只能和xtrabackup --prepare选项一起使用。
	--innodb-miscellaneous：从My.cnf文件读取的一组Innodb选项。以便xtrabackup以同样的配置启动内置的Innodb。通常不需要显示指定。
	--log-copy-interval=#：这个选项指定了log拷贝线程check的时间间隔（默认1秒）。
	--log-stream：xtrabakcup不拷贝数据文件，将事务日志内容重定向到标准输出直到--suspend-at-end文件被删除。这个选项自动开启--suspend-at-end。
	--no-defaults：不从任何选项文件中读取任何默认选项,必须在命令行第一个选项。
	--databases=#：指定了需要备份的数据库和表。
	--database-file=#：指定包含数据库和表的文件格式为databasename1.tablename1为一个元素，一个元素一行。
	--parallel=#：指定备份时拷贝多个数据文件并发的进程数，默认值为1。
	--prepare：xtrabackup在一份通过--backup生成的备份执行还原操作，以便准备使用。
	--print-default：打印程序参数列表并退出，必须放在命令行首位。
	--print-param：使xtrabackup打印参数用来将数据文件拷贝到datadir并还原它们。
	--rebuild_indexes：在apply事务日志之后重建innodb辅助索引，只有和--prepare一起才生效。
	--rebuild_threads=#：在紧凑备份重建辅助索引的线程数，只有和--prepare和rebuild-index一起才生效。
	--stats：xtrabakcup扫描指定数据文件并打印出索引统计。
	--stream=name：将所有备份文件以指定格式流向标准输出，目前支持的格式有xbstream和tar。
	--suspend-at-end：使xtrabackup在--target-dir目录中生成xtrabakcup_suspended文件。在拷贝数据文件之后xtrabackup不是退出而是继续拷贝日志文件并且等待知道xtrabakcup_suspended文件被删除。这项可以使xtrabackup和其他程序协同工作。
	--tables=name：正则表达式匹配database.tablename。备份匹配的表。
	--tables-file=name：指定文件，一个表名一行。
	--target-dir=DIRECTORY：指定backup的目的地，如果目录不存在，xtrabakcup会创建。如果目录存在且为空则成功。不会覆盖已存在的文件。
	--throttle=#：指定每秒操作读写对的数量。
	--tmpdir=name：当使用--print-param指定的时候打印出正确的tmpdir参数。
	--to-archived-lsn=LSN：指定prepare备份时apply事务日志的LSN，只能和xtarbackup --prepare选项一起用。
	--user-memory = #：通过--prepare prepare备份时候分配多大内存，目的像innodb_buffer_pool_size。默认值100M如果你有足够大的内存。1-2G是推荐值，支持各种单位(1MB,1M,1GB,1G)。
	--version：打印xtrabackup版本并退出。
	--xbstream：支持同时压缩和流式化。需要客服传统归档tar,cpio和其他不允许动态streaming生成的文件的限制，例如动态压缩文件，xbstream超越其他传统流式/归档格式的的优点是，并发stream多个文件并且更紧凑的数据存储（所以可以和--parallel选项选项一起使用xbstream格式进行streaming）。

**xtrabackup大部分常用参数都和innobackupex差不多，大家可以自己去官网上看说明，最重要的一点是xtraback支持MyISAM的备份。**

#### 应用场景：

1. 普通全量备份、还原：

		#备份：
		1：指定--defaults-file
		xtrabackup --defaults-file=/etc/mysql/my.cnf --user=root --password=123  --backup --target-dir=/home/zhoujy/xtrabackup/
		
		2：用--datadir取代--defaults-file
		xtrabackup --user=root --password=123  --backup --datadir=/var/lib/mysql/ --target-dir=/home/zhoujy/xtrabackup/
		
		#还原：
		1：(关闭mysql)先prepare
		xtrabackup --prepare --target-dir=/home/zhoujy/xtrabackup/
		
		2：再copy
		rsync -avrP /home/zhoujy/xtrabackup/* /var/lib/mysql/
		
		3：改权限、启动
		chown -R mysql.mysql *

2. 普通增量备份、还原 

		#备份，这里指定几个库和表，也可以是所有库
		1：库全量备份
		xtrabackup --user=root --password=123 --datadir=/var/lib/mysql/ --backup --parallel=3 --databases="xtra_test dba_test" --target-dir=/home/zhoujy/xtrabackup/
		
		2：增量备份
		xtrabackup --user=root --password=123 --datadir=/var/lib/mysql/ --backup --parallel=3 --databases="xtra_test dba_test" --target-dir=/home/zhoujy/xtrabackup1/ --incremental-basedir=/home/zhoujy/xtrabackup/
		
		注意：要是有多个增量备份，第2个增量需要指定第一个增量的目录。和innobackupex一样。
		
		3：还原
		#先prepare全备
		xtrabackup --prepare --apply-log-only --target-dir=/home/zhoujy/xtrabackup/
		#再prepare增量备份
		xtrabackup --prepare --apply-log-only --target-dir=/home/zhoujy/xtrabackup/ --incremental-dir=/home/zhoujy/xtrabackup1/
		
		4：最后prepare 全备
		xtrabackup --prepare --target-dir=/home/zhoujy/xtrabackup/
		
		5：最后copy、改权限。
		
		另外说一个指定表的备份：
		和innobackupex一样，用--databases=dbname.tablename和--tables-file，也可以用--tables（--include），支持正则。
		如备份t开头的数据库下的所有表：
		xtrabackup --user=root --password=123 --datadir=/var/lib/mysql/ --backup --parallel=3 --tables="^t[.]*.*" --target-dir=/home/zhoujy/xtrabackup/

3. 打包压缩备份，注意：--compress不能和--stream=tar一起使用

		##压缩备份
		xtrabackup --user=root --password=123 --datadir=/var/lib/mysql/ --backup --parallel=3 --compress --compress-threads=5 --databases="xtra_test dba_test" --target-dir=/home/zhoujy/xtrabackup/
		
		#解压，在perpare之前需要安装qpress
		for f in `find ./ -iname "*\.qp"`; do qpress -dT2 $f  $(dirname $f) && rm -f $f; done 
		
		#prepare
		xtrabackup --prepare --target-dir=/home/zhoujy/xtrabackup/
		
		#copy，改权限
		
		##打包备份，compress不支持tar。
		xtrabackup --user=root --password=123 --datadir=/var/lib/mysql/ --backup --parallel=3 --compress --compress-threads=5 --stream=xbstream --target-dir=/home/zhoujy/xtrabackup/ >/home/zhoujy/xtrabackup/alldb.xbstream
		
		#解包
		xbstream -x < alldb.xbstream 
		
		#解压
		for f in `find ./ -iname "*\.qp"`; do qpress -dT2 $f  $(dirname $f) && rm -f $f; done 
		
		#prepare
		xtrabackup --prepare --target-dir=/home/zhoujy/xtrabackup
		
		#copy，改权限
		
		##第三方压缩备份：
		 xtrabackup --user=root --password=123 --datadir=/var/lib/mysql/ --backup --parallel=3 --stream=tar --target-dir=/home/zhoujy/xtrabackup/ | gzip /home/zhoujy/xtrabackup/alldb.tar.gz
		
		#解压：
		tar izxvf alldb.tar.gz
		
		#prepare
		xtrabackup --prepare --target-dir=/home/zhoujy/xtrabackup
	
		#copy，改权限

4. 加密备份

		#压缩加密全量备份所有数据库
		1：生成加密key：
		openssl rand -base64 24
		把Key写到文件：
		echo -n "Ue2Wp6dIDWszpI76HQ1u57exyjAdHpRO" > keyfile 
		2：压缩加密全量备份
		xtrabackup --user=root --password=123 --datadir=/var/lib/mysql/ --backup --no-timestamp --compress --compress-threads=3 --encrypt=AES256 --encrypt-key-file=/home/zhoujy/keyfile ----encrypt-threads=3 --parallel=5 --target-dir=/home/zhoujy/xtrabackup/
		
		#还原
		1：解密
		for i in `find . -iname "*\.xbcrypt"`; do xbcrypt -d --encrypt-key-file=/home/zhoujy/keyfile --encrypt-algo=AES256 < $i > $(dirname $i)/$(basename $i .xbcrypt) && rm $i; done
		
		2：解压
		for f in `find ./ -iname "*\.qp"`; do qpress -dT2 $f  $(dirname $f) && rm -f $f; done 
		
		3：prepare
		xtrabackup --prepare --target-dir=/home/zhoujy/xtrabackup/
		
		4：copy，改权限
		rsync -avrP /home/zhoujy/xtrabackup/* /var/lib/mysql/
		chown -R mysql.mysql *

5. 复制环境中的备份：一般生产环境大部分都是主从模式，主提供服务，从提供备份。

		说明：备份 5个线程备份2个数据库，并且文件xtrabackup_slave_info记录GTID和change的信息
		#备份
		xtrabackup --user=root --password=123 --datadir=/var/lib/mysql/ --backup --no-timestamp --slave-info --safe-slave-backup --parallel=5 --databases='xtra_test dba_test' --target-dir=/home/zhoujy/xtrabackup/
		
		#还原
		1：prepare
		xtrabackup --prepare --target-dir=/home/zhoujy/xtrabackup/
		
		2：copy，改权限

**总结：**关于更多的innobackupex的备份可以看[官方文档](https://www.percona.com/doc/percona-xtrabackup/2.4/xtrabackup_bin/xtrabackup_binary.html)和[xtrabackup 安装使用](http://www.cnblogs.com/zhoujinyi/p/4088866.html)，参数可以参考本文上面的介绍说明，通过上面的几个说明看到xtrabackup可以实现：全量、增量、压缩、打包、加密备份，并且支持多线程的备份，并且也提供了长查询超过阀值自动kill的方法，大大提升备份效率。

最后，明白[Percona XtraBackup备份的原理](http://www.cnblogs.com/zhoujinyi/p/5888271.html)之后，根据自己备份场景的特点，选择合理的参数，定制适合自己需求的备份脚本进行工作。