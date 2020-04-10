---
layout: post
title: "Jenkins 持续集成整理 (一）Jenkins 部署"
date: 2018-03-12 17:07:07 +0800
category: jenkins
tags: [jenkins]
---
* content
{:toc}

# 1. 环境准备

- 安装git

	```
	$ yum -y install git
	```

- 安装Java环境

	> 下载地址 [链接](https://www.oracle.com/java/technologies/javase-jdk8-downloads.html)
	
	解压
	
	```
	$ tar zxf jdk-8u102-linux-x64.tar.gz -C /usr/local
	
	```
	
	添加环境变量
	
	```
	$ vim /etc/profile
	export JAVA_HOME=/usr/local/jdk1.8.0_102
	export JRE_HOME=${JAVA_HOME}/jre
	export CLASSPATH=.:${JAVA_HOME}/lib:${JRE_HOME}/lib
	export PATH=${JAVA_HOME}/bin:$PATH
	
	$ source /etc/profile
	
	```
	
	添加软连接
	
	```
	$ln -s $JAVA_HOME/bin/java /usr/bin/java
	
	$ java -version
	java version "1.8.0_102"
	Java(TM) SE Runtime Environment (build 1.8.0_102-b14)
	Java HotSpot(TM) 64-Bit Server VM (build 25.102-b14, mixed mode)
	```
	
# 2. 安装Jenkins

- 安装

	```
	$ wget https://pkg.jenkins.io/redhat/jenkins-2.156-1.1.noarch.rpm
	
	$ rpm -ivh jenkins-2.156-1.1.noarch.rpm
	
	```

- 启动服务

	```
	#重载服务（由于前面修改了Jenkins启动脚本）
	$ systemctl daemon-reload
	
	#启动Jenkins服务
	$ systemctl start jenkins
	
	#将Jenkins服务设置为开机启动
	#由于Jenkins不是Native Service，所以需要用chkconfig命令而不是systemctl命令
	$ /sbin/chkconfig jenkins on
	
	```
	
	检查端口
	
	```
	$ ss -luntp |grep 8080
	```
	
	浏览器输入 `http://<ip addr>:8080`

# 3. 配置Nginx

添加 nginx vhost

```
#新增Jenkins专用Nginx配置文件
$ vi /etc/nginx/conf.d/jenkins.conf

#输入以下内容并保存
server {
    listen       80;        #监听80端口
    server_name  jenkins.ken.io; #监听的域名
    access_log  /var/log/nginx/jenkins.access.log main;
    error_log  /var/log/nginx/jenkins.error.log error;

    location / {            #转发或处理
        proxy_pass http://127.0.0.1:8080; 
    }
    error_page   500 502 503 504  /50x.html;#错误页
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
}
```


# 4. Jenkins国内加速

清华大学下载地址:
[https://mirrors.tuna.tsinghua.edu.cn/jenkins/redhat/](https://mirrors.tuna.tsinghua.edu.cn/jenkins/redhat/)

在jenkins 插件管理页面，将 `https://mirrors.tuna.tsinghua.edu.cn/jenkins/updates/update-center.json` 替换为官方的地址，然后修改  `/var/lib/jenkins/updates/default.json` 文件.

```
# 修改 /var/lib/jenkins/updates/default.json 文件
$ sed -i 's/http:\/\/updates.jenkins-ci.org\/download/https:\/\/mirrors.tuna.tsinghua.edu.cn\/jenkins/g' /var/lib/jenkins/updates/default.json && sed -i 's/http:\/\/www.google.com/https:\/\/www.baidu.com/g' /var/lib/jenkins/updates/default.json

# 重启jenkins
$ systemctl restart jenkins
```