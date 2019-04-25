---
layout: post
title: "redis cluster 整理 (三) 安装报错总结"
date: 2018-03-14 00:21:50 +0800
category: redis
tags: [redis]
---
* content
{:toc}

# redis集群 安装报错总结

### ruby 编译安装

	wget https://cache.ruby-lang.org/pub/ruby/2.5/ruby-2.5.0.tar.gz
	tar zxf ruby-2.5.0.tar.gz
	cd ruby-2.5.0
	./configure --prefix=/usr/local/ruby
	make
	make install
	
### 执行 ./redis-trib.rb 报错

	Traceback (most recent call last):
        2: from /usr/local/bin/redis-trib.rb:25:in `<main>'
        1: from /usr/local/ruby/lib/ruby/2.5.0/rubygems/core_ext/kernel_require.rb:59:in `require'
	/usr/local/ruby/lib/ruby/2.5.0/rubygems/core_ext/kernel_require.rb:59:in `require': cannot load such file -- redis (LoadError)

**解决方法：**

下载 安装ruby ，执行gem install redis来安装ruby执行redis相关依赖	
### gem install redis 报错

#### 错误1 缺少zlib

	ERROR:  Loading command: install (LoadError)
	        cannot load such file -- zlib
	ERROR:  While executing gem ... (NoMethodError)
	    undefined method `invoke_with_build_args' for nil:NilClass


**解决方法：**

	yum -y install zlib-devel
	# 进入ruby源码包
	cd ruby-2.5.0/ext/zlib
	ruby ./extconf.rb
	make

**make报错**

	make: *** No rule to make target `/include/ruby.h', needed by `zlib.o'.  Stop.

**解决方法：**

修改 `Makefile`

将 `zlib.o: $(top_srcdir)/include/ruby.h` 中的 `$(top_srcdir)` 修改为`../..`
或 在 `Makefile ` 最开始给 `top_srcdir`赋值 `top_srcdir = ../..`


#### 错误2 缺少openssl

	ERROR:  While executing gem ... (Gem::Exception)
	    Unable to require openssl, install OpenSSL and rebuild Ruby (preferred) or use non-HTTPS sources

	
**解决方法：**

	yum -y install openssl-devel
	# 进入ruby源码包
	cd ruby-2.5.0/ext/openssl
	ruby ./extconf.rb
	make

**make 报错**

	compiling openssl_missing.c
	make: *** No rule to make target `/include/ruby.h', needed by `ossl.o'.  Stop.
	
**解决方法：**

修改 `Makefile`

同上，将 `${top_srcdir}` 变量赋值`top_srcdir = ../..`

### 执行 ` redis-trib.rb create` 报错`ERR Slot 0 is already busy (Redis::CommandError)`

**解决方法：**

进入每个实例初始化cluster

	redis-cli -h 10.211.55.6 -p 7000
	master:7000> flushall
	master:7000> cluster reset




	
	

