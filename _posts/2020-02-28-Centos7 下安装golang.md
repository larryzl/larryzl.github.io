---
layout: post
title: "Centos7 下安装golang"
date: 2020-02-28 11:14:16 +0800
category: go 
tags: [go]
---
* content
{:toc}


1. 下载安装包

	[下载地址]( https://studygolang.com/dl)
	
	本次下载 `go1.12.5.linux-amd64.tar.gz`

	```
	wget https://studygolang.com/dl/golang/go1.12.5.linux-amd64.tar.gz
	```

2. 解压到指定目录

	`tar -C /usr/local -xzf go1.12.5.linux-amd64.tar.gz`

	解压后在目录 /usr/local/go中

3. 配置环境变量

	设置GOPATH 目录 
	
	```
	mkdir -p /home/gocode
	```
	
	go命令依赖一个重要的环境变量：`$GOPATH`

	GOPATH允许多个目录，当有多个目录时，请注意分隔符，多个目录的时候Windows是分号;，Linux系统是冒号: 
	当有多个GOPATH时默认将go get获取的包存放在第一个目录下 
`$GOPATH`目录约定有三个子目录

	src存放源代码(比如：`.go` `.c` `.h` `.s`等) 
	pkg编译时生成的中间文件（比如：`.a`） 
	bin编译后生成的可执行文件（为了方便，可以把此目录加入到 PATH变量中，如果有多个`gopath`，那么使用`PATH`变量中，如果有多个`gopath`，那么使用`{GOPATH//://bin:}/bin`添加所有的`bin`目录）

	编辑环境 
	`vim /etc/profile` 
	在最后一行加入 按i插入
	
	```
	export GOROOT=/usr/local/go #设置为go安装的路径
	export GOPATH=/home/gocode #默认安装包的路径
	export PATH=$PATH:$GOROOT/bin:$GOPATH/bin
	```
	
	保存后执行 使环境生效 
	`source /etc/profile `
	
4. 一键安装脚本
	
	```
	#!/bin/bash
	set -e
	
	GOROOT=/home/go
	GOPATH=/user/local/go
	
	echo $(date +"[%Y-%m-%d %H:%M:%S]") 开始下载 go 1.12.5
	wget https://studygolang.com/dl/golang/go1.12.5.linux-amd64.tar.gz
	
	echo $(date +"[%Y-%m-%d %H:%M:%S]") 下载完成，开始解压
	tar zxf go1.12.5.linux-amd64.tar.gz -C /usr/local/
	
	mkidr $GOROOT
	
	cat >> /etc/profile <<EOF
	export GOROOT=$GOROOT
	export GOPATH=$GOPATH
	export PATH=\$PATH:\$GOROOT/bin:\$GOPATH/bin
	EOF
	source /etc/profile
	echo $(date +"[%Y-%m-%d %H:%M:%S]") Golang 环境设置完成
	go env
	```