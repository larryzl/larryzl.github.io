---
layout: post
title: "CentOS6 + Nginx + uwsgi + django + Python"
date: 2017-6-09 14:56:58 +0800
category: Django
tags: [Django]
---
* content
{:toc}

### 配置网关


	#vi /etc/sysconfig/network
	NETWORKING=yes(表示系统是否使用网络，一般设置为yes。如果设为no，则不能使用网络，而且很多系统服务程序将无法启动)
	HOSTNAME=centos(设置本机的主机名，这里设置的主机名要和/etc/hosts中设置的主机名对应)
	GATEWAY=192.168.1.1(设置本机连接的网关的IP地址。例如，网关为10.0.0.2)
	

### 配置DNS


	#vi /etc/resolv.conf
	nameserver: 114.114.114.114 # IBM哥们讲的国内比较快的一个DNS

### 配置epel源

	wget http://dl.fedoraproject.org/pub/epel/6/i386/epel-release-6-8.noarch.rpm# 下载
	chmod +x epel-release-6-8.noarch.rpm# 权限
	rpm -ivh epel-release-6-8.noarch.rpm# 安装
	yum update

### 安装pip

	wget https://raw.github.com/pypa/pip/master/contrib/get-pip.py
	wget https://raw.githubusercontent.com/pypa/pip/master/contrib/get-pip.py
	python get-pip.py# 没有install

### pip 安装django

	pip install django==1.6.6
	
### pip 安装mysqldb

	yum install mysql-devel mysql mysql-server python-devel
	pip install MySQL-python
	
### 修改mysql的root密码

	mysql -uroot
	use mysql;#有的版本默认管理数据库不是这个名字（*注意）
	
	MySQL> update user set password=PASSWORD('newpassword') where User='root';
	MySQL> flush privileges; //刷新权限，重启mysql也可以
	MySQL> quit

### 配置nginx

	http {    
		include       /etc/nginx/mime.types;
	    default_type  application/octet-stream;
	    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '                      						 '$status $body_bytes_sent "$http_referer" '
	                     '"$http_user_agent" "$http_x_forwarded_for"';
	   	access_log  /var/log/nginx/access.log  main;
	  	sendfile        on;
	   	#tcp_nopush     on;
		#keepalive_timeout  0;
		keepalive_timeout  65;
		#gzip  on;     
		# Load config files from the /etc/nginx/conf.d directory    
		# The default server is in conf.d/default.conf    
		include /etc/nginx/conf.d*.conf;
		#-----------------------------------------------------
		
	    client_max_body_size 1024m;    
	    server {        
	    	listen 80;        
	    	server_name 192.168.46.5;        
	    	location / {            
	    		uwsgi_pass 127.0.0.1:9000;            
	    		include uwsgi_params;            
	    		uwsgi_param UWSGI_CHDIR /data/www/helloworld;            
	    		uwsgi_param UWSGI_SCRIPT django_wsgi;            
	    		access_log off;        
	    	}        
	    	location ^~ /static {
	       	root /data/www/xtyw/;        
	   		}        
	   		location ~* ^.+\.(png|mpg|avi|mp3|swf|zip|tgz|gz|rar|bz2|doc|docx|xls|exe|ppt|t|tar|mid|midi|wav|rtf|,peg)$ {
	       	root /data/medias;
	       	access_log off;
	   	    }    
		}
	
### 安装uwsgi

	pip install uwsgi
	
### 配置uwsgi

进入项目主目录，即settings.py所在目录，创建uwsgi.ini配置文件

	[uwsgi]
	socket = 0.0.0.0:9000
	master = true
	pidfile = /etc/nginx/uwsgi.pid
	processes = 4
	chdir = /data/www/xtyw/
	wsgi-file = /data/www/xtyw/xtyw/wsgi.py
	profiler = true
	memory-report = true
	enable-threads = true
	logdate = true
	limit-as = 6048
	daemnize = /data/logs/django.log

执行：

	#uwsgi uwsgi.ini
	如果是xml文件
	#uwsgi -x uwsgi.xml
	
