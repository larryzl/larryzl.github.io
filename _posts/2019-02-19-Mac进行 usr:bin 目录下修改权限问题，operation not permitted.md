---
layout: post
title: "[转] Mac进行 usr/bin 目录下修改权限问题，operation not permitted"
date: 2019-02-19 11:37:14 +0800
category: 其他
tags: [其他]
---
* content
{:toc}


一般情况下我们在使用mac系统过程中下载一些文件、新建一些项目之后，这些文件都会默认是只读状态，这时我们只需要简单的一句权限设置命令就可以解决

	sudo chmod -R 777 你要修改文件上层目录的路径
	
但是我们在对 `usr/bin` 目录下的文件进行操作的时候往往会出现这样的错误（上一篇博客中关于git安装进行软连接就会遇到这样的问题）

	operation not permitted

这是因为一些mac用户在升级系统之后，电脑启用了SIP（System Integrity Protection），增加了rootless机制，导致即使在root权限下依然无法修改文件，在必要时候为了能够修改下面的文件，我们只能关闭该保护机制

1. 重启，过程中按住 command+R，进入保护模式

2. 打开terminal终端，输入

		csrutil disable
	
3. 再次重启，即可对 usr/bin 目录下文件进行修改

**PS：如果要恢复保护机制，重新进入保护模式，输入
csrutil enable**