---
layout: post
title: "CentOS 7 安装 PostgreSQL 9.5"
date: 2018-09-28 11:23:56 +0800
category: PostgreSQL
tags: [PostgreSQL，db]
---
* content
{:toc}

> 操作系统版本：Linux localhost.localdomain 3.10.0-327.18.2.el7.x86_64 #1 SMP Thu May 12 11:03:55 UTC 2016 x86_64 x86_64 x86_64 GNU/Linux

> 数据库版本： psql (PostgreSQL) 9.5.3

### 安装PostgreSQL

安装过程参考官方文档，地址列于此，[Linux downloads (Red Hat family) 。](https://www.postgresql.org/download/linux/redhat/)

CentOS Yum 工具安装，简单方便，查看了一下官方源版本，显示目前最新版本是9.2.15,需要更新源，文档中有专门的rpm包列表，[RPM LIST](https://yum.postgresql.org/repopackages.php)。

1. 添加RPM
	
		yum install https://download.postgresql.org/pub/repos/yum/9.5/redhat/rhel-7-x86_64/pgdg-centos95-9.5-2.noarch.rpm

2. 安装PostgreSQL 9.5

		 yum install postgresql95-server postgresql95-contrib

3. 初始化数据库

		 /usr/pgsql-9.5/bin/postgresql95-setup initdb

4. 启动，设置开机启动

		systemctl enable postgresql-9.5.service
		systemctl start postgresql-9.5.service
		
自此，PostgreSQL 9.5 安装完成，此过程中注意安装权限，我在安装过程中一直使用的是root用户进行的安装。

### 配置PostgreSQL

> PostgreSQL 安装完成后，会建立一下‘postgres’用户，用于执行PostgreSQL，数据库中也会建立一个'postgres'用户，默认密码为自动生成，需要在系统中改一下。

1. 修改用户密码

		# 切换用户
		su - postgres
		# 登录数据库
		psql -U postgres 
		# 修改用户密码
		ALTER USER postgres WITH PASSWORD 'abc123'
		# 退出数据库
		\q

2. 开启远程连接

		vi /var/lib/pgsql/9.5/data/postgresql.conf 
		
		# 修改listen_addresses
		listen_addresses = '*'
		
3. 信任远程连接
		
	pg_hba.conf每条记录声明一种联接类型，一个客户端 IP 地址范围（如果和联接类型相关的话），一个数据库名，一个用户名字，以及对匹配这些参数的联接使用的认证方法。
	
		# 修改认证文件 pg_hba.conf
		vi /var/lib/pgsql/9.5/data/pg_hba.conf
		
		local   all             all                                     trust
		# IPv4 local connections:
		host    all             all             127.0.0.1/32            trust
		# IPv6 local connections:
		host    all             all             10.20.0.0/16            trust
		host    all             all             ::1/128                 ident

	联接使用的认证方法：
	
	- trust:
	
		无条件地允许联接。这个方法允许任何可以与PostgreSQL数据库服务器联接的用户以他们期望的任意PostgreSQL 数据库用户身份进行联接，而不需要口令。
		
	- md5
	
		要求客户端提供一个 MD5 加密的口令进行认证。
		
	如果我想让10.86.12.0~10.86.12.154的IP段能访问PostgreSQL 数据库，需要增加下面一行：
	
		host   all             all           10.86.12.0/24                  trust
	

4. 打开防火墙

	 CentOS 防火墙中内置了PostgreSQL服务，配置文件位置在/usr/lib/firewalld/services/postgresql.xml，我们只需以服务方式将PostgreSQL服务开放即可。
	 
	 	firewall-cmd --add-service=postgresql --permanent  #开放postgresql服务
    	firewall-cmd --reload  #重载防火墙
    	
5. 重启数据库

		systemctl restart postgresql-9.5.service
