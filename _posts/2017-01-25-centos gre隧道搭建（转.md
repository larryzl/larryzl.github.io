---
layout: post
title: "centos gre隧道搭建（转）"
date: 2017-01-25 11:24:30 +0800
category: Linux
tags: [gre]
---
* content
{:toc}


Aliyun:

	ip tunnel add gre1 mode gre remote 218.*.*.*  local 172.19.*.* ttl 255   <===218*.*.*是APN-TEST的公网IP  172.*.*.*是阿里云主机内网IP
	ip link set gre1 up
	ip addr add 1.1.1.1/32 peer 1.1.1.2/32 dev gre1


APN-TEST:

	ip tunnel add gre1 mode gre remote 47.*.*.* local 172.168.*.* ttl 255    <===47.*.*.*是阿里云主机的公网IP  172.*.*.*是APN-TEST内网IP
	ip link set gre1 up
	ip addr add 1.1.1.1/32 peer 1.1.1.2/32 dev gre1

 

CentOS默认不加载gre内核模块

	# modprobe ip_gre   临时加载gre模块（重启后失效）
	
	# lsmod |grep gre 进行确认

 

开机自添加模块方法

	# cat /etc/sysconfig/modules/ip_gre.modules 
	#!/bin/sh 
	/sbin/modinfo -F filename ip_gre > /dev/null 2>&1 
	if [ $? -eq 0 ]; then 
	    /sbin/modprobe ip_gre
	fi
	
	# chmod 755  /etc/sysconfig/modules/ip_gre.modules

 

一旦加载gre模块，该模块会自动生成gre0核gretap0这两个虚拟接口。不用管它。也删除不了。

想要删除的话必须卸载gre模块（modprobe -r  ip_gre）
