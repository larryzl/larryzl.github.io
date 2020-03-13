---
layout: post
title: "MySQL 清理bin-log"
date: 2018-10-31 14:15:45 +0800
category: MySQL
tags: [MySQL]
---
* content
{:toc}

1. 查看binlog日志

		mysql> show binary logs;
		
		+------------------+------------+
		| Log_name         | File_size  |
		+------------------+------------+
		| mysql-bin.000061 |      50624 |
		| mysql-bin.000062 |       5159 |
		| mysql-bin.000063 |        126 |
		| mysql-bin.000064 |       3067 |
		| mysql-bin.000065 |        503 |
		| mysql-bin.000066 |        494 |
		| mysql-bin.000067 |        107 |
		| mysql-bin.000068 |       1433 |
		| mysql-bin.000069 |       7077 |
		| mysql-bin.000070 |        107 |
		| mysql-bin.000071 |        804 |
		| mysql-bin.000072 |       7642 |
		| mysql-bin.000073 |       2198 |
		| mysql-bin.000074 |     350139 |
		| mysql-bin.000075 |        126 |
		| mysql-bin.000076 |      51122 |
		| mysql-bin.000077 | 1074279197 |
		| mysql-bin.000078 | 1074435879 |
		| mysql-bin.000079 |  928917122 |
		+------------------+------------+

2. 删除某个日志文件之前的所有日志文件

		mysql> purge binary logs to 'mysql-bin.000079';  
		mysql> show binary logs;
		
		+------------------+-----------+
		| Log_name         | File_size |
		+------------------+-----------+
		| mysql-bin.000079 | 928917122 |
		+------------------+-----------+
		
		
		mysql> reset master;   #重置所有的日志

3. 关闭mysql的binlog日志

		#log-bin=mysql-bin  在my.cnf里面注释掉binlog日志
		
		重启mysql
		
		service mysqld restar
