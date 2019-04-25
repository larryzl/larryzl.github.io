---
layout: post
title: "Linux 配置 zsh 以及 oh-my-zsh"
date: 2018-03-14 00:27:13 +0800
category: Linux
tags: [Linux]
---
* content
{:toc}


### Zsh 安装

CentOS 安装：

	sudo yum install -y zsh

Ubuntu 安装：

	sudo apt-get install -y zsh
	
在检查下系统的 shell：

	cat /etc/shells

你会发现多了一个：/bin/zsh

### 使用 Zsh 扩展集合：oh-my-zsh

- oh-my-zsh 帮我们整理了一些常用的 Zsh 扩展功能和主题：https://github.com/robbyrussell/oh-my-zsh
- 我们无需自己去捣搞 Zsh，直接用 oh-my-zsh 就足够了，如果你想继续深造的话那再去弄。
- 先安装 git：`sudo yum install -y git`
- 安装 oh-my-zsh（这个过程可能会有点慢，或者需要重试几次）：

		wget https://raw.github.com/robbyrussell/oh-my-zsh/master/tools/install.sh -O - | sh
- 在以 root 用户为前提下，oh-my-zsh 的安装目录：

		/root/.oh-my-zsh
- 在以 root 用户为前提下，Zsh 的配置文件位置：

		/root/.zshrc
- 为 root 用户设置 zsh 为系统默认 shell：

		chsh -s /bin/zsh root

- 如果你要重新恢复到 bash：

		chsh -s /bin/bash root

- 现在你关掉终端或是重新连上 shell，现在开头是一个箭头了



### CentOS zsh&oh-my-zsh 安装脚本


	#!/bin/bash
	#set -e
	#
	# Desc: CentOS zsh 一键安装脚本
	# CreateDate: 2019-03-08 17:56:49
	# LastModify:
	# Author: larry
	#
	# History:
	#
	#
	# ---------------------- Script Begin ----------------------
	#
	
	#- 查看当前shell
	echo "当前 shell 为:  $SHELL"
	
	#- 安装zsh
	echo "正在安装 zsh"
	yum -y install git wget zsh
	
	#- 设置默认shell
	echo "正在设置默认shell"
	chsh -s /bin/zsh
	
	#- 安装oh-my-zsh（自动）
	sh -c "$(curl -fsSL https://raw.githubusercontent.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"
	
	# 修改主题
	sed -i 's/ZSH_THEME=.*$/ZSH_THEME="ys"/g' ~/.zshrc
	sed -i 's/^plugins=.*$/plugins=(git autojump zsh-completions systemd yum wd common-aliases git-flow grails rvm history-substring-search github gradle svn node npm zsh-syntax-h ighlighting sublime)/g' ~/.zshrc


### 插件推荐：

#### wd
- 简单地讲就是给指定目录映射一个全局的名字，以后方便直接跳转到这个目录，比如：
- 编辑配置文件，添加上 wd 的名字：vim /root/.zshrc
- 我常去目录：/opt/setups，每次进入该目录下都需要这样：cd /opt/setups
- 现在用 wd 给他映射一个快捷方式：cd /opt/setups ; wd add setups
- 以后我在任何目录下只要运行：wd setups 就自动跑到 /opt/setups 目录下了
- 插件官网：https://github.com/mfaerevaag/wd


#### autojump
- 这个插件会记录你常去的那些目录，然后做一下权重记录，你可以用这个命令看到你的习惯：j --stat，如果这个里面有你的记录，那你就只要敲最后一个文件夹名字即可进入，比如我个人习惯的 program：j program，就可以直接到：/usr/program
- 插件官网：https://github.com/wting/autojump
- 官网插件下载地址：https://github.com/wting/autojump/downloads
- 插件下载：wget https://github.com/downloads/wting/autojump/autojump_v21.1.2.tar.gz
- 解压：tar zxvf autojump_v21.1.2.tar.gz
- 进入解压后目录并安装：cd autojump_v21.1.2/ ; ./install.sh
- 再执行下这个：source /etc/profile.d/autojump.sh
- 编辑配置文件，添加上 autojump 的名字：vim /root/.zshrc


#### zsh-syntax-highlighting
- 这个插件会对终端命令高亮显示,比如正确的拼写会是绿色标识,否则是红色,另外对于一些shell输出语句也会有高亮显示,算是不错的辅助插件
- 插件官网：https://github.com/zsh-users/zsh-syntax-highlighting
- 安装，复制该命令：'git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting'
- 编辑：vim ~/.zshrc，找到这一行，后括号里面的后面添加：plugins=( 前面的一些插件名称 zsh-syntax-highlighting)
- 刷新下配置：source ~/.zshrc