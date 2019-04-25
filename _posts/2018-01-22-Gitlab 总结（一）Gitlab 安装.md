---
layout: post
title: "Gitlab 总结（一）Gitlab 安装"
date: 2018-01-22 13:37:47 +0800
category: git
tags: [git]
---
* content
{:toc}



>GitLab 是一个用于仓库管理系统的开源项目，使用Git作为代码管理工具，并在此基础上搭建起来的web服务。安装方法是参考GitLab在GitHub上的Wiki页面。
>
>GitLab是由GitLabInc.开发，使用MIT许可证的基于网络的Git仓库管理工具，且具有wiki和issue跟踪功能。使用Git作为代码管理工具，并在此基础上搭建起来的web服务。
GitLab由乌克兰程序员DmitriyZaporozhets和ValerySizov开发，它由Ruby写成。后来，一些部分用Go语言重写。截止2018年5月，该公司约有290名团队成员，以及2000多名开源贡献者。GitLab被IBM，Sony，JülichResearchCenter，NASA，Alibaba，Invincea，O’ReillyMedia，Leibniz-Rechenzentrum(LRZ)，CERN，SpaceX等组织使用

**GitLab功能：**

1. 代码托管服务
2. 访问权限控制
3. 问题跟踪，bug的记录和讨论
4. 代码审查，可以查看、评论代码
5. 社区版基于 MIT License开源完全免费

## GitLab硬件需求

1. CPU

	1核心的CPU，基本上可以满足需求，大概支撑100个左右的用户，不过在运行GitLab网站的同时，还需要运行多个worker以及后台job，显得有点捉襟见肘了。

	2核心的CPU是推荐的配置，大概能支撑500个用户.
	
	4核心的CPU能支撑 2,000 个用户.
	
	8核心的CPU能支撑 5,000 个用户.
	
	16核心的CPU能支撑 10,000 个用户.
	
	32核心的CPU能支撑 20,000 个用户.
	
	64核心的CPU能支持多达 40,000 个用户.
	
2. Memory

	你需要至少4GB的可寻址内存（RAM交换）来安装和使用GitLab！操作系统和任何其他正在运行的应用程序也将使用内存，因此请记住，在运行GitLab之前，您至少需要4GB的可用空间。使用更少的内存GitLab将在重新配置运行期间给出奇怪的错误，并在使用过程中发生500个错误.
	
	1GBRAM + 3GB of swap is the absolute minimum but we strongly adviseagainst this amount of memory. See the unicorn worker section belowfor more advice.
	
	2GBRAM + 2GB swap supports up to 100 users but it will be very slow
	
	4GBRAM isthe recommended memory size for all installations and supportsup to 100 users
	
	8GBRAM supports up to 1,000 users
	
	16GBRAM supports up to 2,000 users
	
	32GBRAM supports up to 4,000 users
	
	64GBRAM supports up to 8,000 users
	
	128GBRAM supports up to 16,000 users
	
	256GBRAM supports up to 32,000 users

	建议服务器上至少有2GB的交换，即使您目前拥有足够的可用RAM。如果可用的内存更改，交换将有助于减少错误发生的机会。

3. Unicorn Workers(进程数) 

	可以增加独角兽工人的数量，这通常有助于减少应用程序的响应时间，并增加处理并行请求的能力.
	
	对于大多数情况，我们建议使用：CPU内核1 =独角兽工人。所以对于一个有2个核心的机器，3个独角兽工人是理想的。
	
	对于所有拥有2GB及以上的机器，我们建议至少三名独角兽工人。如果您有1GB机器，我们建议只配置两个Unicorn工作人员以防止过度的交换.

4. Database

	- PostgreSQL

	- MySQL/MariaDB

	强烈推荐使用PostgreSQL而不是MySQL/ MariaDB，因为GitLab的所有功能都不能与MySQL/ MariaDB一起使用。例如，MySQL没有正确的功能来以有效的方式支持嵌套组.

	运行数据库的服务器应至少有5-10 GB的可用存储空间，尽管具体要求取决于GitLab安装的大小

5. PostgreSQL要求

	从GitLab 9.0起，PostgreSQL 9.2或更新版本是必需的，不支持早期版本。

6. Redis and Sidekiq

	Redis存储所有用户会话和后台任务队列。Redis的存储要求最低，每个用户大约25kB。

	Sidekiq使用多线程进程处理后台作业。这个过程从整个Rails堆栈（200MB）开始，但是由于内存泄漏，它可以随着时间的推移而增长。在非常活跃的服务器（10,000个活跃用户）上，Sidekiq进程可以使用1GB的内存。

7. Prometheus and its exporters

	从Omnibus GitLab 9.0开始，默认情况下，Prometheus及其相关出口商启用，可以轻松，深入地监控GitLab。这些进程将使用大约200MB的内存，具有默认设置。这个还可以监控k8s

8. Node exporter

	节点导出器允许您测量各种机器资源，如内存，磁盘和CPU利用率。默认端口9100

9. Redis exporter

   Redis出口商允许您测量各种Redis指标。

10. Postgres exporter

	Postgres导出器允许您测量各种PostgreSQL度量。

11. GitLab monitor exporter

	GitLab监视器导出器允许您测量各种GitLab指标。

12. Supported web browsers

	支持Firefox，Chrome /Chromium，Safari和Microsoft浏览器（Microsoft Edge和Internet Explorer 11）的当前和之前的主要版本。

## 安装GitLab

1. 准备

	- 关闭SELinux
	
			 sed -i's/^SELINUX=.*/#&/;s/^SELINUXTYPE=.*/#&/;/SELINUX=.*/aSELINUX=disabled' /etc/sysconfig/selinux
	
	- 永久修改下主机名，需要重启系统之后生效
	
		- Redhat6中修改
			
				root@git ~]# vi /etc/sysconfig/network
		
				NETWORKING=yes
		
				HOSTNAME=git.server.com #修改成你自己的主机名
			
		- Redhat7中修改
				
				[root@noede1 ~]# vi /etc/hostname
				
				gitlab.server.com
				
				#永久修改
				
				[root@git ~]#hostnamectl set-hostname gitlab.server.com
				
				#添加域名
				
				[root@git ~]#cat /etc/hosts
				
				192.168.201.131 gitlab.server.com


	
2. 官网安装方法

		#- 安装依赖、开启端口

		sudo yum install -y curl policycoreutils-python openssh-server openssh-clients cronie
		
		sudo lokkit -s http -s ssh

	-
 	
		#- 安装邮件服务、设置开机启动

		sudo yum install postfix

		sudo service postfix start
		
		sudo chkconfig postfix on

	-

		#- 添加GitLab仓库到yum源,并用yum方式安装到服务器上

		curl -sS http://packages.gitlab.com.cn/install/gitlab-ce/script.rpm.sh | sudo bash
		sudo EXTERNAL_URL="http://gitlab.example.com" yum install -y gitlab-ce
		
		#- EXTERNAL_URL是设置用什么域名访问你的gitlab，此时也可以直接yum install gitlab-ce。安装完成后在修改配置文件/etc/gitlab/gitlab.rb

	-
	
		#- 启动gitlab，就可以访问你的gitlab了。
		
		sudo gitlab-ctl start
		
		sudo gitlab-ctl reconfigure
		
		
3. rpm 安装方法

	一般直接yum安装，是不会成功的。因为地址被强了，当然如果你可以配置代理，你可以成功。

	那么我们用手动下载RPM包方式，安装gitlab。

		#- 下载gitlab并安装
		wget https://packages.gitlab.com/gitlab/gitlab-ce/packages/el/6/gitlab-ce-10.7.0-ce.0.el6.x86_64.rpm
		
		rpm -ivh gitlab-ce-10.7.0-ce.0.el6.x86_64.rpm
		warning: gitlab-ce-10.7.0-ce.0.el6.x86_64.rpm: Header V4 RSA/SHA1 Signature, key ID f27eab47: NOKEY
		Preparing...                ########################################### [100%]
		   1:gitlab-ce              ########################################### [100%]
		It looks like GitLab has not been configured yet; skipping the upgrade script.
		
		       *.                  *.
		      ***                 ***
		     *****               *****
		    .******             *******
		    ********            ********
		   ,,,,,,,,,***********,,,,,,,,,
		  ,,,,,,,,,,,*********,,,,,,,,,,,
		  .,,,,,,,,,,,*******,,,,,,,,,,,,
		      ,,,,,,,,,*****,,,,,,,,,.
		         ,,,,,,,****,,,,,,
		            .,,,***,,,,
		                ,*,.
		  
		
		
		     _______ __  __          __
		    / ____(_) /_/ /   ____ _/ /_
		   / / __/ / __/ /   / __ `/ __ \
		  / /_/ / / /_/ /___/ /_/ / /_/ /
		  \____/_/\__/_____/\__,_/_.___/
		  
		
		Thank you for installing GitLab!
		GitLab was unable to detect a valid hostname for your instance.
		Please configure a URL for your GitLab instance by setting `external_url`
		configuration in /etc/gitlab/gitlab.rb file.
		Then, you can start your GitLab instance by running the following command:
		  sudo gitlab-ctl reconfigure
		
		For a comprehensive list of configuration options please see the Omnibus GitLab readme
		https://gitlab.com/gitlab-org/omnibus-gitlab/blob/master/README.md

	
	-

		#- 也可以用其他源下载RPM包,速度会快一些。
		wget  https://mirrors.tuna.tsinghua.edu.cn/gitlab-ce/yum/el6/gitlab-ce-10.7.0-ce.0.el6.x86_64.rpm
		
		rpm -ivh gitlab-ce-10.7.0-ce.0.el6.x86_64.rpm

	-
	
		#- 如果速度都比较慢，报超时错误。你可以先下载到电脑上，手动传到服务器。

		#- 然后启动gitlab，并访问
		
		sudo gitlab-ctl reconfigure
		
		#- 还需要修改EXTERNAL_URL

4. 总结

	rpm安装方法比较方便，

## GitLab 中文版


首先查看当前gitlab版本

	$ cat /opt/gitlab/embedded/service/gitlab-rails/VERSION  
	
**本文介绍8.x 9.x 10.x 中文版汉化安装**


1. 8.x 汉化安装 [项目地址](https://gitlab.com/larryli/gitlab.git)

		#- 安装依赖软件
		
		yum -y install git

		
		#- clone 中文项目

		git clone https://gitlab.com/larryli/gitlab.git Gitlab-cn && cd Gitlab-cn
		
		#- 备份/opt/gitlab/embedded/service目录下的gitlab-rails目录，该目录下的内容主要是web应用部分
		
		cp -rf/opt/gitlab/embedded/service/gitlab-rails{,.ori}
		
		#- 关闭gitlab这个服务
		
		gitlab-ctl stop
		
		#- 开始汉化
		
		cp -rf Gitlab-cn/* /opt/gitlab/embedded/service/gitlab-rails/
		
		#— 测试是否汉化成功
		
		gitlab-ctl start
		
		
2. 9.x 汉化安装

		#- 安装依赖软件
		
		yum -y install git patch
		
		#- clone 项目
		
		git clone https://gitlab.com/xhang/gitlab.git
	
		#- 生成补丁
		
		cd gitlab/
		git diff v9.1.2 v9.1.2-zh> tmp/9.1.2-zh.diff
		
		#- 更新补丁到gitlab中
		
		patch -d/opt/gitlab/embedded/service/gitlab-rails -p1 < /tmp/9.1.2-zh.diff
		
		#- 重新配置gitlab
		
		gitlab-ctl reconfigure
		

3. 10.x 汉化安装 [gitlab项目地址](https://gitlab.com/xhang/gitlab)

		#- 安装依赖软件
		
		yum -y install git patch

	-
	
		#- clone 中文项目
		
		git clone https://gitlab.com/xhang/gitlab.git /usr/local/src/gitlab
		
	-
	
		#- 停止gitlab
		
		gitlab-ctl stop
		
	-
	
		#- 获取gitlab汉化包
		
		cd /usr/local/src/gitlab
		git diff origin/10-7-stable origin/10-7-stable-zh > /tmp/10.7.diff
		
	-

		#- 更新汉化补丁到gitlab中
		
		cd /tmp/
		patch -d/opt/gitlab/embedded/service/gitlab-rails -p1 < 10.7.diff
		
	-
	
		#- 重新配置gitlab
		
		gitlab-ctl reconfigure
		gitlab-ctl start
		
	



		
		