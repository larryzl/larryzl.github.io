---
layout: post
title: "Linux 命令总结之 echo"
date: 2016-03-07 17:50:12 +0800
category: Linux命令
tags: [Linux命令]
---
* content
{:toc}

## 简介

echo是一种最常用的与广泛使用的内置于Linux的bash和C shell的命令，通常用在脚本语言和批处理文件中来在标准输出或者文件中显示一行文本或者字符串。

## 语法

echo [选项][字符串]

### echo 选项列表

|选项|	描述|
|---|---|
|-n|不输出末尾的换行符。|
|-e	|启用反斜线转义。|
|\b	|退格|
|\\	|反斜线|
|\n|新行|
|\r|回车|
|\t	|水平制表符|
|\v|垂直制表符|

实例1：设置echo命令彩色输出

	[0m: 正常
	[1m: 粗体
	[4m: 字体加上下划线
	[7m: 逆转前景和背景色
	[8m: 不可见字符
	[9m: 跨行字体
	[30m: 灰色字体
	[31m: 红色字体
	[32m: 绿色字体
	[33m: 棕色字体
	[34m: 蓝色字体
	[35m: 紫色字体
	[36m: 浅蓝色字体
	[37m: 浅灰字体
	[38m: 黑色字体
	[40m: 黑色背景
	[41m: 红色背景
	[42m: 绿色背景
	[43m: 棕色背景
	[44m: 蓝色背景
	[45m: 紫色背景
	[46m: 浅蓝色背景
	[47m: 浅灰色背景

下面的命令将使用红色打印输出：

	[root@Finish ~]# echo -e "\033[31mLinux\033[0m"
	Linux

实例2：-n选项的使用

	[root@Finish ~]# echo -n "who are you"
	who are you[root@Finish ~]#

实例3：-e选项的使用

选项‘-e‘扮演了转义字符反斜线的翻译器

	[root@Finish ~]# echo -e "who \bare \byou"
	whoareyou

‘-e‘后带上'\b'会删除字符间的所有空格

	[root@Finish ~]# echo -e "\vwho \vare \vyou"
	 
	who
	    are
	        you

‘-e‘后面跟上‘\v’会加上垂直制表符