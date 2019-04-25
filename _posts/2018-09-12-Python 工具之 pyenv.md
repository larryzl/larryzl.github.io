---
layout: post
title:  "Python 工具之 pyenv"
date:   2018-09-12 11:06:43 +0800
categories: Python
tags: Python
---

* content
{:toc}


## pyenv 安装

1. 安装依赖包:

		yum install zlib-devel bzip2-devel openssl-devel ncurses-devel sqlite-devel readline-devel tk-devel gdbm-devel db4-devel libpcap-devel xz-devel -y

2. 安装pyenv包:

		git clone https://github.com/pyenv/pyenv.git ~/.pyenv

3. 设置环境变量:

		#vim ~/.bashrc
		export PYENV_ROOT="$HOME/.pyenv"
		export PATH="$PYENV_ROOT/bin:$PATH"
		eval "$(pyenv init -)"

		#source ~/.bashrc
		#即是启动语句，重启系统执行这条语句
		exec $SHELL


4. 查看pyenv支持的python版本,同时也是检验有没有安装成功:

		pyenv install --list

5. 查看当前pyenv可检测到的所有版本,处于激活状态的版本前以 * 标示.

		[root@localhost ~]# pyenv versions
		  system
		  3.5.1
		* 3.5.3 (set by /root/.pyenv/version)

6. 查看当前处于激活状态的版本,括号中内容表示这个版本是由哪条途径激活的（global、local、shell）

		# pyenv version
		3.6.0 (set by /Data/scripts/py/.python-version)

7. 将3.5.1作为全局变量,使用如下命令.

		[root@localhost ~]# pyenv global 3.5.1
		[root@localhost ~]# pyenv version
		3.5.1 (set by /root/.pyenv/version)

8. 设置面向程序的本地版本,通过将版本号写入当前目录下的.python-version 文件的方式。

	**在本地创建目录ops，执行pyenv local 3.5.3后，只有在这个目录是3.5.3的版本，别的目录使用默认的版本.**

		[root@localhost ~]# python -V
		Python 3.5.1
		[root@localhost ~]# pyenv versions
		  system
		* 3.5.1 (set by /root/.pyenv/version)
		  3.5.3
		[root@localhost ~]#
		[root@localhost ~]# mkdir ops
		[root@localhost ~]# cd ops/
		[root@localhost ops]# pyenv local 3.5.3
		[root@localhost ops]# python -V
		Python 3.5.3
		[root@localhost ops]# cd ..
		[root@localhost ~]# python -V
		Python 3.5.1

9. 安装你需要的Python版本(如3.4.0):

		pyenv install 3.4.0 -v

		#小技巧，可以在/root/.pyenv/目录下创建cache目录，将下载好的Python-3.4.0的包放在该目录下，就可以直接执行安装，而不需要下载,节省下载时间.

10. 安装完成之后需要对数据库进行更新：

		pyenv rehash

11. 卸载python 3.4.0版本.

		pyenv uninstall 3.4.0

## pyenv 本地安装包

事先下载之，放到~/.pyenv/cache目录即可。修改~/.pyenv/plugins/python-build/share/python-build/3.5.2文件，

	cat ~/.pyenv/plugins/python-build/share/python-build/3.5.2
	#require_gcc
	install_package "openssl-1.0.2g" "https://www.openssl.org/source/openssl-1.0.2g.tar.gz#b784b1b3907ce39abf4098702dade6365522a253ad1552e267a9a0e89594aa33" mac_openssl --if has_broken_mac_openssl
	install_package "readline-6.3" "http://ftpmirror.gnu.org/readline/readline-6.3.tar.gz#56ba6071b9462f980c5a72ab0023893b65ba6debb4eeb475d7a563dc65cafd43" standard --if has_broken_mac_readline
	if has_tar_xz_support; then
	  install_package "Python-3.5.2" "~/.pyenv/cache/Python-3.5.2.tar.gz" ldflags_dirs standard verify_py35 ensurepip
	else
	  install_package "Python-3.5.2" "~/.pyenv/cache/Python-3.5.2.tar.gz" ldflags_dirs standard verify_py35 ensurepip
	fi

由于没有~/.pyenv/cache目录，进行手工创建，

	$ mkdir ~/.pyenv/cache

如果使用手工安装，则需要安装一些依赖，

	yum install -y gcc make patch gdbm-devel openssl-devel sqlite-devel zlib-devel bzip2-devel readline-devel

需要事先准备好Python-3.5.2.tar.gz的安装包，放到~/.pyenv/cache目录下。然后，在命令行直接使用pyenv install 3.5.2即可，

	 pyenv install 3.5.2

安装完毕，使用version命令进行查看

	pyenv version
	3.5.2 (set by /home/lavenliu/.python-version)