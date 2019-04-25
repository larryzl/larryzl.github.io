---
layout: post
title: "MySQL 主从整理（二） 使用xtrabackup在线搭建主从"
date: 2018-01-03 15:48:06 +0800
category: MySQL
tags: [MySQL]
---
* content
{:toc}



## 1. 说明

### 1.1 xtrabackup

mysqldump对于导出10G以下的数据库或几个表，还是适用的，而且更快捷。一旦数据量达到100-500G，无论是对原库的压力还是导出的性能，mysqldump就力不从心了。Percona-Xtrabackup备份工具，是实现MySQL在线热备工作的不二选择，可进行全量、增量、单表备份和还原。（但当数据量更大时，可能需要考虑分库分表，或使用 LVM 快照来加快备份速度了）

2.2版本 xtrabackup 能对InnoDB和XtraDB存储引擎的数据库非阻塞地备份，innobackupex通过perl封装了一层xtrabackup，对MyISAM的备份通过加表读锁的方式实现。2.3版本 xtrabackup 命令直接支持MyISAM引擎。

**XtraBackup优势 ：**

1. 无需停止数据库进行InnoDB热备

2. 增量备份MySQL

3. 流压缩到传输到其它服务器

4. 能比较容易地创建主从同步

5. 备份MySQL时不会增大服务器负载

### 1.2 replication

