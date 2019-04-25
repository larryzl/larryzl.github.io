---
layout: post
title: "MySQL 备份整理(五）xtrabackup基础介绍"
date: 2017-12-24 16:35:51 +0800
category: MySQL
tags: [MySQL]
---
* content
{:toc}


> XtraBackup 是由[Percona](https://www.percona.com/software/mysql-database/percona-xtrabackup)开发的一个用于MySQL数据库物理热备份的开源软件，支持MySQL（Oracle）、Percona Server和MariaDB。 XtraBackup主要特点有：
> 
> 1. 非阻塞备份InnoDB等事物引擎数据库
> 2. 备份MyISAM表会阻塞（需要锁）
> 3. 支持全备、增量备份、压缩备份
> 4. 快速增量备份
> 5. Percona支持归档redo log的备份
> 6. Percona 5.6+ 支持轻量级的backup-lock替代原来重量级的FTWRL，此时即使备份非事务引擎表也不会阻塞InnoDB的DML语句了
> 7. 支持加密备份、流备份（备份到远程主机）、并行本地备份、并行压缩、并行加密、并行应用备份期间产生的redo日志、并行copy-back
> 8. 支持不发备份，只备份某个库，某个表
> 9. 支持不发恢复
> 10. 支持备份单个表分区
> 11. 支持备份速度限制，指备份产生的IO速度的限制
> 12. 支持point-in-time恢复
> 13. 支持compat备份，也即使不备份索引数据，索引在prepare时`--rebuild-indexs`
> 14. 支持备份buffer pool
> 15. 支持单表export，import到其他库
> 16. 支持rsync来缩短备份非实物引擎表的锁定时间


## InnoDB的备份原理

> 在InnoDB内部会维护一个redo日志文件，我们也可以叫做事物日志文件。事物日志会存储每一个InnoDB表数据的记录修改。当InnoDB启动时，InnoDB会检查数据文件和事务日志，并执行两个步骤： 它应用（前滚）已经提交的事务日志到数据文件，并将修改过但没有提交的数据进行回滚操作。
> 



- innobackupex 的基本流程如下：

	1. 开启redo日志拷贝线程，从最新的检查点开始顺序拷贝redo日志；
	2. 开启idb文件拷贝线程，拷贝innodb表的数据；
	3. idb文件拷贝结束，通知调用FTWRL，获取一致性位点；
	4. 备份非innodb表（系统表）和frm文件
	5. 由于此时没有新事务提交，等待redo日志拷贝完成；
	6. 最新的redo日志拷贝完成后，相当于此时的innodb表和非innodb表数据都是最新的；
	7. 获取binlog位点，此时数据库的状态是一致的。
	8. 释放锁，备份结束。


- XtraBackup的改进

	无论是muysqldump，还是innobackupex备份工具，为了获取一致性位点，都强依赖于FTWRL。这个锁的杀伤力非常大，因为持有锁的这段时间，整个数据库实质上不能对外提供写服务。此外，犹豫FTWRL需要关闭表，如有大查询，会导致FTWRL等待，进而导致DML堵塞的时间变长。即使是备库，备份很慢，那么持有锁的时间就会很长。即使全部是innodb表，也会因为有mysql库系统表存在，导致会锁定一定的时间。

	为了解决这个问题，Percona公司对MySQL的Server层做了改进，引入了BACKUP LOCK。具体而言，通过'LOCK TABLES FOR BACKUP‘命令来获取一致性数据（包括非innodb表）；通过“LOCK BINLOG FOR BACKUP”获取一致性位点，尽量减少因为数据库备份带来的服务受损。

- 与FTWRL的区别

	- LOCK TABLES FOR BACKUP

		作用： 备份数据


		1. 禁止非innodb表更新
		2. 禁止所有表的ddl
		
		优化点：  

		1. 不会被大查询堵塞
		2. 不会堵塞innodb表的读取和更新，这点非常重要，对于业务表全部是innodb的情况，则备份过程中农DML完全不受损

	- LOCK BINLOG FOR BACKUP

		作用： 获取一致性位点
		
		1. 禁止对位点更新的操作

		优化点：

		1. 允许DDL和更新，知道写binlog为止。

- release历史

	2.4.1 支持MySQL5.7(5.7.10)
	
	2.3.2 命令行语法跟随MySQL5.6的变化而变化。另外命令行支持--datadir
	
	2.3.1 innobackupex脚本用c重写，并且只是xtrabackup的符号连接。innobackupex支持2.2版本所有的特性，但是目前已降级在下个Major版本中移除，innobackupex将不支持所有新特性的语法，同时xtrabackup现在支持MyISAM的拷贝并且支持innobakcupex的所有特性。innobackupex先前特性的语法xtrabackup同样支持
	
	2.2.21 支持5.6（基于5.6.24版本）
	
	2.2.8 基于5.6.22 （解决当总redo log超过4G，prepare会失败的问题）
	
	2.2.6 通过show variables读取Mysql选项。在初始化表扫描的时候输出更详细信息
	
	2.2.5 基于5.6.21
	
	2.2.1 移除xtrabackup_56 xtrabakcup_55,只保留xtrabakcup.移除Build脚本，支持cmake编译。基于5.6.16
	
	2.1.6 innobackupex --force-non-empty-directories
	
	2.1.4 MySQL versions 5.1.70, 5.5.30, 5.6.11 
	
	innobackupex --no-lock ,拷贝非Innodb数据时不停止复制线程，但是条件是备份期间非事务型表上不能有DDL或者DML操作
	
	innobackupex --decrypt and innobackupex --decompress,
	
	2.1.1 支持紧凑备份，加密备份。不在支持5.0内置Innodb和5.1内置Innoddb。移除--remote-host选项
	
	2.1.0 支持mysql5.6的所有特性（GTID, 可移动表空间，独立undo表空间，5.6样式的buffer pool导出文件）
	
	支持5.6引入的innodb buffer pool预载。buffer pool dumps可以生成或者导入加速启动。在备份时buffer pool dump拷贝到备份目录，在还原阶段拷贝回data目录，
	
	--log-copy-interval 可配置log拷贝线程检查的间隔时间
	
	如果开启gtid，xtrabackup_binlog_info储存gtid的值
	
	支持xtrabackup --export，这个选项生成5.6样式的元数据文件。可以通过alter table import tablespace导入
	
	2.0.5 --defaults-extra-file 存备份用户的用户名和密码的配置文件
	
	2.0.3 支持--move-back
	
	1.9.1 支持压缩备份，之前能能streaming备份之后通过外部工具压缩
	
	支持streaming增量备份
	
	LRU DUMP
	
	1.6.4 innobackupex支持--rsync选项 在datadir目录进行两阶段rsync（首先没有写锁，之后有写锁，）减少写锁持有的时间



## XtraBackup 安装

> XtraBackup 分3种安装方式：
> 
> 1. RPM安装
> 2. YUM安装
> 3. 源码安装
> 
> 下面分别介绍下3种安装方式



### RPM 安装

1. 安装依赖

		$ yum -y install libev numactl

2. 官网下载[Percona XtraBackup](https://www.percona.com/downloads/XtraBackup/LATEST/)的相应版本的RPM包

		$ wget https://www.percona.com/downloads/XtraBackup/Percona-XtraBackup-2.4.9/binary/redhat/6/x86_64/percona-xtrabackup-24-2.4.9-1.el6.x86_64.rpm

		$ rpm -ivh percona-xtrabackup-24-2.4.9-1.el6.x86_64.rpm

### YUM 安装

1. 安装percona源

		$ yum install http://www.percona.com/downloads/percona-release/redhat/0.1-4/percona-release-0.1-4.noarch.rpm

2. yum安装xtrabackup

		$ yum -y install percona xtrabackup

### 源码安装

编译及软件依赖


- centos5/6 需要升级cmake至2.8.2版本以上，解决： 安装cmake 3.4.3版本测试通过
- centos5 gcc g++ 需要升级gcc至4.4 以上，解决： 安装4.4.7 测试通过



1. 安装依赖

		$ yum install cmake gcc gcc-c++ libaio libaio-devel automake autoconf \
		  bison libtool ncurses-devel libgcrypt-devel libev-devel libcurl-devel \
		  vim-common

2. 下载[GitHub](https://github.com/percona/percona-xtrabackup/tree/release-2.4.9)源码

		$ git clone https://github.com/percona/percona-xtrabackup.git
		$ cd percona-xtrabackup
		$ git checkout 2.4.9

3. 编译安装
	
	**2.4.4之前版本安装方法**

		$ cmake -DBUILD_CONFIG=xtrabackup_release -DWITH_MAN_PAGES=OFF && make -j4
		$ make install
		# 如果不指定默认安装目录 /usr/local/xtrabackup
		#如果要指定安装目录，
		$ make DESTDIR=... install
		# 或者
		$ cmake -DINSTALL_LAYOUT=...
		
	**2.4.4之后版本安装方法**
	
	需要指定`boost`安装目录
	
	1. 安装`boost`

			wget http://downloads.sourceforge.net/project/boost/boost/1.59.0/boost_1_59_0.tar.gz 
			tar zxf boost_1_59_0.tar.gz
			cd boost_1_59_0
			./bootstrap.sh
			./b2 install --perfix=dir
	
	2. 安装xtrabackup

			cmake -DDOWNLOAD_BOOST=1  -DWITH_BOOST=/usr/local/boost_1_59_0 -DBUILD_CONFIG=xtrabackup_release -DWITH_MAN_PAGES=OFF && make -j4
			make install
			
4. 配置环境变量

	在 `/etc/profile`文件末尾添加

		export PATH=$PATH:/usr/local/xtrabackup/bin

	查看版本

		$xtrabackup --version
		xtrabackup version 2.4.9 based on MySQL server 5.7.13 Linux (x86_64) (revision id: )
		