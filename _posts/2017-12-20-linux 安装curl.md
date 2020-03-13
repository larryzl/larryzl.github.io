---
layout: post
title: "在Linux下安装curl"
date: 2018-12-20 17:10:28 +0800
category: Linux命令
tags: [Linux命令]
---
* content
{:toc}


### 安装步骤

1. 下载curl源码包， 在[官网](http://curl.haxx.se/download/)下载

	选择需要安装的版本下载
	
		wget https://curl.haxx.se/download/curl-7.20.0.tar.gz
	
2. 解压安装

		tar zxf curl-7.20.0.tar.gz
		
3. 备份&覆盖安装
	
		mv /usr/bin/curl /usr/bin/curl_bak
		cd curl-7.20.0
		./configure
		make
		make install

4. 使用`curl --version` 查看是否安装成功


		curl --version
		curl 7.20.0 (x86_64-unknown-linux-gnu) libcurl/7.20.0 OpenSSL/1.0.2k zlib/1.2.7
		Protocols: dict file ftp ftps http https imap imaps pop3 pop3s rtsp smtp smtps telnet tftp 
		Features: IPv6 Largefile NTLM SSL libz 


	查看到正常版本返回结果，安装完成
