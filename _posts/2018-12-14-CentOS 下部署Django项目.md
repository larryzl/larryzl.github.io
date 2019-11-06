---
layout: post
title: "CentOS 下部署Django项目"
date: 2018-12-14 11:49:41 +0800
category: Python,Django
tags: [Python,Django]
---
* content 
{:toc}

# CentOS 下部署Django项目


## 系统准备

```
#!/bin/bash
#
# 更新基础环境
#
############

yum update -y
yum -y groupinstall "Development tools"
yum install openssl-devel bzip2-devel expat-devel gdbm-devel readline-devel sqlite-devel psmisc libffi-devel


# 下载Python3
cd /usr/local/src/
wget https://www.python.org/ftp/python/3.6.6/Python-3.6.6.tgz
tar -zxvf Python-3.6.6.tgz
cd Python-3.6.6
./configure --prefix=/usr/local/python3
make
make install
ln -s /usr/local/python3/bin/python3.6 /usr/bin/python3
ln -s /usr/local/python3/bin/pip3.6 /usr/bin/pip3

# 测试安装情况
python3 -V
pip3 -V

# 安装virtualenv 方便项目管理
pip3 install virtualenv
ln -s /usr/local/python3/bin/virtualenv /usr/bin/virtualenv


```

## 创建virtualenv环境

脚本名称： `create_webdir.sh`

使用方法： `./create_webdir.sh web_dirname`


```
#!/bin/bash

# 创建项目目录
if [ "${1}0" == "0" ];then
	echo "dir name is null"
	echo "Usage:"
	echo "./create_webdir.sh web_dirname"
	exit 1
fi
web_dirname=$1
mkdir $web_dirname

#创建虚拟环境目录
virtualenv --python=/usr/bin/python3 venv
source ${web_dirname}/venv/bin/activate

# 安装django uwsgi
pip3 install django 
pip3 install uwsgi

#留意：uwsgi要安装两次，先在系统里安装一次，然后进入对应的虚拟环境安装一次。
#给uwsgi建立软链接，方便使用

ln -s /usr/local/python3/bin/uwsgi /usr/bin/uwsgi

```

## 创建线上Django环境

```
#!/bin/bash
django-admin.py startproject mysite

```

## 配置uwsgi

在网站项目根目录创建`mysite.xml`文件

```
<uwsgi>    
   <socket>127.0.0.1:8997</socket> <!-- 内部端口，自定义 --> 
   <chdir>/data/wwwroot/mysite/</chdir> <!-- 项目路径 -->            
   <module>mysite.wsgi</module>  <!-- mysite为wsgi.py所在目录名--> 
   <processes>4</processes> <!-- 进程数 -->     
   <daemonize>uwsgi.log</daemonize> <!-- 日志文件 -->
</uwsgi>
```

注意<module>里的mysite，为wsgi.py所在的目录名。

## 安装nginx和配置nginx.conf文件

```
server {
    listen 80;
    server_name  www.django.cn; #改为自己的域名，没域名修改为127.0.0.1:80
    charset utf-8;
    location / {
       include uwsgi_params;
       uwsgi_pass 127.0.0.1:8997;  #端口要和uwsgi里配置的一样
       uwsgi_param UWSGI_SCRIPT mysite.wsgi;  #wsgi.py所在的目录名+.wsgi
       uwsgi_param UWSGI_CHDIR /data/wwwroot/mysite/; #项目路径
       
    }
    location /static/ {
    alias /data/wwwroot/mysite/static/; #静态资源路径
    }
}

```

## 启动uwsgi

```
uwsgi -x mysite.xml
```

