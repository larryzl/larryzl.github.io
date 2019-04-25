---
layout: post
title: "Gitlab 总结（二）Gitlab 配置"
date: 2018-01-23 10:54:30 +0800
category: git
tags: [git]
---
* content
{:toc}

> 上一篇介绍gitlab社区版的安装与汉化，这篇主要介绍gitlab基础使用与常用配置

**gitlab 常用命令**

语法:

	gitlab-ctl command (subcommand)
	
	
|命令|说明|
|---|---|
|start|启动所有服务|
|stop|关闭所有服务|
|restart|重启所有服务|
|status|查看所有服务状态|
|tail|查看日志信息|
|service-list|列举所有启动服务|
|graceful-kill|平稳停止一个服务|


1. gitlab 添加邮件功能

		[root@vm1 ~]# vim /etc/gitlab/gitlab.rb
		[root@vm1 ~]# grep -P "^[^#].*smtp_|user_email|gitlab_email" /etc/gitlab/gitlab.rb
		gitlab_rails['gitlab_email_enabled'] = true
		gitlab_rails['gitlab_email_from'] = 'username@domain.cn'
		gitlab_rails['gitlab_email_display_name'] = 'Admin'
		gitlab_rails['gitlab_email_reply_to'] = 'usernamei@domain.cn'
		gitlab_rails['gitlab_email_subject_suffix'] = '[gitlab]'
		gitlab_rails['smtp_enable'] = true
		gitlab_rails['smtp_address'] = "smtp.exmail.qq.com"
		gitlab_rails['smtp_port'] = 25 
		gitlab_rails['smtp_user_name'] = "username@domain.cn"
		gitlab_rails['smtp_password'] = "password"
		gitlab_rails['smtp_domain'] = "domain.cn"
		gitlab_rails['smtp_authentication'] = "login"
		gitlab_rails['smtp_enable_starttls_auto'] = true
		gitlab_rails['smtp_tls'] = false
		user['git_user_email'] = "username@domain.cn"
		
		[root@vm1 ~]# gitlab-ctl reconfigure
		[root@vm1 ~]# gitlab-ctl restart

	使用gitlab-rails console命令进行发送邮件测试，如下所示：

		[root@vm1 ~]# gitlab-rails console 
		Loading production environment (Rails 4.2.10)
		irb(main):001:0>  Notify.test_email('user@destination.com', 'Message Subject', 'Message Body').deliver_now
		
		Notify#test_email: processed outbound mail in 2219.5ms
		
		Sent mail to user@destination.com (2469.5ms)
		Date: Fri, 04 May 2018 15:50:10 +0800
		From: Admin <username@domain.cn>
		Reply-To: Admin <username@domain.cn>
		To: user@destination.com
		Message-ID: <5aec10b24cfaa_93933fee282db10c162d@vm1.mail>
		Subject: Message Subject
		Mime-Version: 1.0
		Content-Type: text/html;
		 charset=UTF-8
		Content-Transfer-Encoding: 7bit
		Auto-Submitted: auto-generated
		X-Auto-Response-Suppress: All
		
		<!DOCTYPE html PUBLIC "-//W3C//DTD HTML 4.0 Transitional//EN" "http://www.w3.org/TR/REC-html40/loose.dtd">
		<html><body><p>Message Body</p></body></html>
		
		=> #<Mail::Message:70291731344240, Multipart: false, Headers: <Date: Fri, 04 May 2018 15:50:10 +0800>, <From: Admin <username@domain.cn>>, <Reply-To: Admin <username@domain.cn>>, <To: user@destination.com>, <Message-ID: <5aec10b24cfaa_93933fee282db10c162d@vm1.mail>>, <Subject: Message Subject>, <Mime-Version: 1.0>, <Content-Type: text/html; charset=UTF-8>, <Content-Transfer-Encoding: 7bit>, <Auto-Submitted: auto-generated>, <X-Auto-Response-Suppress: All>>
		irb(main):002:0>quit
		

2. gitlab的使用

	在浏览器中输入 http://192.168.60.119/ ，然后 change password: ，并使用root用户登录 即可 (后续动作根据提示操作)


	修改密码也可以：
	
		gitlab-rails console production
		irb(main):001:0> user = User.where(id: 1).first // id为1的是超级管理员
		irb(main):002:0>user.password = 'yourpassword' // 密码必须至少8个字符
		irb(main):003:0>user.save! // 如没有问题 返回true
		exit // 退出
	
	
	gitlab相关配置参数修改 `/etc/gitlab/gitlab.rb`
	
	
3. gitlab备份与还原

	执行备份：
	
		gitlab-rake gitlab:backup:create

	会在备份目录（默认`/var/opt/gitlab/backups/`）生成一个备份文件，如`1537856454_gitlab_backup.tar`，其中1537856454即为此次备份都版本号。
	
	执行还原：
	
		gitlab-rake gitlab:backup:restore BACKUP=备份版本号

4. gitlab升级

	> 注意：由于升级不能跨越大版本号，因此只能升级到当前大版本号到最高版本，方可升级到下一个大版本号

	依次执行下面指令逐步升级，在每一步安装成功后如果发现界面500，不可访问，那么执行```gitlab-ctl reconfigure```指令刷新配置文件。（一定保证数据可以正常访问方可执行下一步升级指令）
	
		yum install gitlab-ce-8.17.8-ce.0.el6
		yum install gitlab-ce-9.5.9-ce.0.el6
		yum install gitlab-ce-10.8.0-ce.0.el6
		yum install gitlab-ce-11.3.0-ce.0.el6
	
	查看当情版本号 

		cat /opt/gitlab/embedded/service/gitlab-rails/VERSION



	
	