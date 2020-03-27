---
layout: post
title: "CentOS7 搭建BIDN9 DNS服务器"
date: 2019-03-26 17:44:33 +0800
category: Linux
tags: [Linux]
---
* content
{:toc}


# 1.安装

## 1.1. 环境


>系统版本: CentOS 7
>
>软件版本: BIND 9.11
>
>IP:  192.168.8.22

## 1.2. 安装

```
yum -y install bind
```


# 2. 配置

## 2.1. 修改全局配置文件


修改 `/etc/named.conf`

修改 `listen-on port 53 { any; };` 将监听所有网卡

修改 `allow-query     { any; };` 接受所有地址请求

```
options {
        listen-on port 53 { any; };
        listen-on-v6 port 53 { ::1; };
        directory       "/var/named";
        dump-file       "/var/named/data/cache_dump.db";
        statistics-file "/var/named/data/named_stats.txt";
        memstatistics-file "/var/named/data/named_mem_stats.txt";
        recursing-file  "/var/named/data/named.recursing";
        secroots-file   "/var/named/data/named.secroots";
        allow-query     { any; };

        recursion yes;

        dnssec-enable yes;
        dnssec-validation yes;

        bindkeys-file "/etc/named.root.key";

        managed-keys-directory "/var/named/dynamic";

        pid-file "/run/named/named.pid";
        session-keyfile "/run/named/session.key";
};

logging {
        channel default_debug {
                file "data/named.run";
                severity dynamic;
        };
};

zone "." IN {
        type hint;
        file "named.ca";
};

include "/etc/named.rfc1912.zones";
include "/etc/named.root.key";
```


## 2.2. 建立本地解析域名

1. 在 `/etc/named.rfc1912.zones` 文件中追加

	```
	zone "test.com" IN {
		type master;
		file "test.com.zone";
		allow-update {none;};
	};
	
	zone "8.168.192.in-addr.arpa" IN {
		type master;
		file "192.168.8.arpa";
		allow-update {none;};
	};
	```

2. 检查是否正确

	`named-checkconf`

3. 建立区域文件

	**正向解析**
	
	```
	$ cat > /var/named/test.com.zone<<EOF
	$TTL 1D
	@    IN 	SOA    test.com. rname.invalid. (
							                    0    ; serial
							                    1D    ; refresh
							                    1H    ; retry
							                    1W    ; expire
							                    3H )    ; minimum
        NS      @
        A       127.0.0.1
        AAAA    ::1
	ns    IN    A    192.168.8.29
	harbor    IN    A    192.168.8.29
	EOF
	```
	
	**反向解析**
		
	```
	$ cat /var/named/192.168.8.arpa
	$TTL 1D
	@    IN SOA    test.com. rname.invalid. (
	                    0    ; serial
	                    1D    ; refresh
	                    1H    ; retry
	                    1W    ; expire
	                    3H )    ; minimum
	    IN    NS    @
	    A    127.0.0.1
	    AAAA    ::1
	    PTR    localhost.
	ns    IN    A    192.168.8.29
	244    IN    PTR    ns.test.com
	250    IN    PTR    harbor.test.com
	```
		
	修改文件属主
	
	```
	chown -R named.named /var/named
	```
	
4. 重启服务

	```
	systemctl restart named
	```
	
# 3. 测试

用另外一台主机 使用`dig`命令测试

```
$ yum -y install bind-utils

$ dig @192.168.8.22 harbor.test.com +short
192.168.8.29
```

测试通过