---
layout: post
title: "redis cluster 整理 (二) 集群部署"
date: 2018-03-12 10:20:09 +0800
category: redis
tags: [redis]
---
* content
{:toc}


## redis cluster 搭建


### 环境准备

搭建redis cluster，准备3台服务器，将master 节点、slave 节点 分布在3台机器上，保证任意一台机器宕机，不影响服务使用，如下表。

|IP|端口|
|---|---|
|Master Node|
|10.211.55.6|7000|
|10.211.55.7|8000|
|10.211.55.8|9000|
||
|Slave Node|
|10.211.55.6|7001|
|10.211.55.7|8001|
|10.211.55.8|9001|
||
|Others Node|
|10.211.55.6|7002|
|10.211.55.7|8002|
|10.211.55.8|9002|


### redis cluster 搭建

1. 编译安装redis

		wget http://download.redis.io/releases/redis-4.0.1.tar.gz
		tar zxf redis-4.0.1.tar.gz
		cd redis-4.0.1
		make
		make install PREFIX=/usr/local/redis
		cp src/redis-trib.rb /usr/local/bin/redis-trib.rb
		cd /usr/local/redis
		mkdir {etc,log,pid,rdb}
		
2. 创建配置文件

		cd /usr/local/redis/etc/
		vi redis_7000.conf
		
		# redis后台运行
		daemonize yes
		# pid文件，运行多个实例时，需要指定不同的pid文件
		pidfile /usr/local/redis/pid/redis_7000.pid
		# 监听端口，运行多个实例时，需要指定不同的断奶口
		port 7000 
		
		tcp-backlog 511
		tcp-keepalive 0
		# 日志等级
		loglevel notice 
		# 日志文件位置
		logfile /usr/local/redis/log/redis_7000.log
		# 可用数据库数
		databases 16
		appendonly no
		# 使用压缩rdb文件，rdb文件压缩使用LZF压缩算法，yes：压缩，但是需要一些cpu的消耗。no：不压缩，需要更多的磁盘空间
		rdbcompression yes
		# 是否校验rdb文件。从rdb格式的第五个版本开始，在rdb文件的末尾会带上CRC64的校验和。这跟有利于文件的容错性，但是在保存rdb文件的时候，会有大概10%的性能损耗，所以如果你追求高性能，可以关闭该配置
		rdbchecksum yes
		# rdb文件的名称
		dbfilename dump_7000.rdb
		# 数据目录，数据库的写入会在这个目录。rdb、aof文件也会写在这个目录
		dir /usr/local/redis/rdb
		
		
3. 按照以上配置创建其他实例配置文件，将端口替换成相应端口

4. 使用redis-trib.rb搭建集群 (遇到错误请看错误总结）

	- 安装ruby

			wget https://cache.ruby-lang.org/pub/ruby/2.5/ruby-2.5.0.tar.gz
			tar zxf ruby-2.5.0.tar.gz
			cd ruby-2.5.0
			./configure --prefix=/usr/local/ruby
			make
			make install

	- gem 安装 redis

			gem install redis
			gem list

	- 创建集群

			redis-trib.rb create --replicas 1 10.211.55.6:7000 10.211.55.7:8000 10.211.55.8:9000 10.211.55.6:7001 10.211.55.7:8001 10.211.55.8:9001
			>>> Creating cluster
			>>> Performing hash slots allocation on 6 nodes...
			Using 3 masters:
			10.211.55.6:7000
			10.211.55.7:8000
			10.211.55.8:9000
			Adding replica 10.211.55.7:8001 to 10.211.55.6:7000
			Adding replica 10.211.55.6:7001 to 10.211.55.7:8000
			Adding replica 10.211.55.8:9001 to 10.211.55.8:9000
			M: d542d12e66bc2e5eec5d1eff04cc6afd31125a55 10.211.55.6:7000
			   slots:0-5460 (5461 slots) master
			M: 8ed63224e33b12308587d6e999a43df69d7ffc24 10.211.55.7:8000
			   slots:5461-10922 (5462 slots) master
			M: 893974f5a56a3ff732e31dcece3a2b37fce421ea 10.211.55.8:9000
			   slots:10923-16383 (5461 slots) master
			S: b1ea02819986fabc3a722201c2d2ddcfcc0916f5 10.211.55.6:7001
			   replicates 8ed63224e33b12308587d6e999a43df69d7ffc24
			S: b7aab7e0ce75ff1c87c8b7cb0b0d2320becbe5b3 10.211.55.7:8001
			   replicates d542d12e66bc2e5eec5d1eff04cc6afd31125a55
			S: 71a79ea31183031040e6527c6605a52ded3e95b0 10.211.55.8:9001
			   replicates 893974f5a56a3ff732e31dcece3a2b37fce421ea
			Can I set the above configuration? (type 'yes' to accept): yes  <----此处输入yes
			>>> Nodes configuration updated
			>>> Assign a different config epoch to each node
			>>> Sending CLUSTER MEET messages to join the cluster
			Waiting for the cluster to join...
			>>> Performing Cluster Check (using node 10.211.55.6:7000)
			M: d542d12e66bc2e5eec5d1eff04cc6afd31125a55 10.211.55.6:7000
			   slots:0-5460 (5461 slots) master
			   1 additional replica(s)
			M: 893974f5a56a3ff732e31dcece3a2b37fce421ea 10.211.55.8:9000
			   slots:10923-16383 (5461 slots) master
			   1 additional replica(s)
			S: b1ea02819986fabc3a722201c2d2ddcfcc0916f5 10.211.55.6:7001
			   slots: (0 slots) slave
			   replicates 8ed63224e33b12308587d6e999a43df69d7ffc24
			S: 71a79ea31183031040e6527c6605a52ded3e95b0 10.211.55.8:9001
			   slots: (0 slots) slave
			   replicates 893974f5a56a3ff732e31dcece3a2b37fce421ea
			M: 8ed63224e33b12308587d6e999a43df69d7ffc24 10.211.55.7:8000
			   slots:5461-10922 (5462 slots) master
			   1 additional replica(s)
			S: b7aab7e0ce75ff1c87c8b7cb0b0d2320becbe5b3 10.211.55.7:8001
			   slots: (0 slots) slave
			   replicates d542d12e66bc2e5eec5d1eff04cc6afd31125a55
			[OK] All nodes agree about slots configuration.
			>>> Check for open slots...
			>>> Check slots coverage...
			[OK] All 16384 slots covered.

		通过输出可以看出 master节点 与slave节点分布
		
			Using 3 masters:
			10.211.55.6:7000
			10.211.55.7:8000
			10.211.55.8:9000
			Adding replica 10.211.55.7:8001 to 10.211.55.6:7000
			Adding replica 10.211.55.6:7001 to 10.211.55.7:8000
			Adding replica 10.211.55.8:9001 to 10.211.55.8:9000

		slots分布

			10.211.55.6:7000 slots:0-5460 (5461 slots) master
			10.211.55.7:8000 slots:5461-10922 (5462 slots) master
			10.211.55.8:9000 slots:10923-16383 (5461 slots) master

		检查集群信息

			$ redis-trib.rb check 10.211.55.6:7000              
			>>> Performing Cluster Check (using node 10.211.55.6:7000)
			M: d542d12e66bc2e5eec5d1eff04cc6afd31125a55 10.211.55.6:7000
			   slots:0-5460 (5461 slots) master
			   1 additional replica(s)
			M: 893974f5a56a3ff732e31dcece3a2b37fce421ea 10.211.55.8:9000
			   slots:10923-16383 (5461 slots) master
			   1 additional replica(s)
			S: b1ea02819986fabc3a722201c2d2ddcfcc0916f5 10.211.55.6:7001
			   slots: (0 slots) slave
			   replicates 8ed63224e33b12308587d6e999a43df69d7ffc24
			S: 71a79ea31183031040e6527c6605a52ded3e95b0 10.211.55.8:9001
			   slots: (0 slots) slave
			   replicates 893974f5a56a3ff732e31dcece3a2b37fce421ea
			M: 8ed63224e33b12308587d6e999a43df69d7ffc24 10.211.55.7:8000
			   slots:5461-10922 (5462 slots) master
			   1 additional replica(s)
			S: b7aab7e0ce75ff1c87c8b7cb0b0d2320becbe5b3 10.211.55.7:8001
			   slots: (0 slots) slave
			   replicates d542d12e66bc2e5eec5d1eff04cc6afd31125a55
			[OK] All nodes agree about slots configuration.
			>>> Check for open slots...
			>>> Check slots coverage...
			[OK] All 16384 slots covered.
		
			$ redis-trib.rb info 10.211.55.6:7000
			10.211.55.6:7000 (d542d12e...) -> 0 keys | 5461 slots | 1 slaves.
			10.211.55.8:9000 (893974f5...) -> 0 keys | 5461 slots | 1 slaves.
			10.211.55.7:8000 (8ed63224...) -> 0 keys | 5462 slots | 1 slaves.
			[OK] 0 keys in 3 masters.
			0.00 keys per slot on average.
			
			$ redis-cli -h 10.211.55.6 -p 7000 cluster info
			cluster_state:ok
			cluster_slots_assigned:16384
			cluster_slots_ok:16384
			cluster_slots_pfail:0
			cluster_slots_fail:0
			cluster_known_nodes:6
			cluster_size:3
			cluster_current_epoch:6
			cluster_my_epoch:1
			cluster_stats_messages_ping_sent:623
			cluster_stats_messages_pong_sent:762
			cluster_stats_messages_sent:1385
			cluster_stats_messages_ping_received:757
			cluster_stats_messages_pong_received:623
			cluster_stats_messages_meet_received:5
			cluster_stats_messages_received:1385

	
		至此，redis cluster 已经安装成功
		



### 节点增减，分配solt

**使用`redis-trib.rb` 增加节点**

	$ redis-trib.rb add-node 10.211.55.6:7002 10.211.55.6:7000
	
	>>> Adding node 10.211.55.6:7002 to cluster 10.211.55.6:7000
	>>> Performing Cluster Check (using node 10.211.55.6:7000)
	S: d542d12e66bc2e5eec5d1eff04cc6afd31125a55 10.211.55.6:7000
	   slots: (0 slots) slave
	   replicates b7aab7e0ce75ff1c87c8b7cb0b0d2320becbe5b3
	M: 893974f5a56a3ff732e31dcece3a2b37fce421ea 10.211.55.8:9000
	   slots:10923-16383 (5461 slots) master
	   1 additional replica(s)
	S: b1ea02819986fabc3a722201c2d2ddcfcc0916f5 10.211.55.6:7001
	   slots: (0 slots) slave
	   replicates 8ed63224e33b12308587d6e999a43df69d7ffc24
	S: 71a79ea31183031040e6527c6605a52ded3e95b0 10.211.55.8:9001
	   slots: (0 slots) slave
	   replicates 893974f5a56a3ff732e31dcece3a2b37fce421ea
	M: 8ed63224e33b12308587d6e999a43df69d7ffc24 10.211.55.7:8000
	   slots:5461-10922 (5462 slots) master
	   1 additional replica(s)
	M: b7aab7e0ce75ff1c87c8b7cb0b0d2320becbe5b3 10.211.55.7:8001
	   slots:0-5460 (5461 slots) master
	   1 additional replica(s)
	[OK] All nodes agree about slots configuration.
	>>> Check for open slots...
	>>> Check slots coverage...
	[OK] All 16384 slots covered.
	>>> Send CLUSTER MEET to node 10.211.55.6:7002 to make it join the cluster.
	[OK] New node added correctly.

**查看集群状态**

	$ redis-cli -h 10.211.55.6 -p 7002 cluster nodes
	b1ea02819986fabc3a722201c2d2ddcfcc0916f5 10.211.55.6:7001@17001 slave 8ed63224e33b12308587d6e999a43df69d7ffc24 0 1552425772916 12 connected
	0081c3db1996613d5cbb6b15910e4653b398a6fa 10.211.55.6:7002@17002 myself,master - 0 1552425771000 0 connected
	b7aab7e0ce75ff1c87c8b7cb0b0d2320becbe5b3 10.211.55.7:8001@18001 master - 0 1552425772000 14 connected 0-5460
	d542d12e66bc2e5eec5d1eff04cc6afd31125a55 10.211.55.6:7000@17000 slave b7aab7e0ce75ff1c87c8b7cb0b0d2320becbe5b3 0 1552425772513 14 connected
	71a79ea31183031040e6527c6605a52ded3e95b0 10.211.55.8:9001@19001 slave 893974f5a56a3ff732e31dcece3a2b37fce421ea 0 1552425772000 13 connected
	893974f5a56a3ff732e31dcece3a2b37fce421ea 10.211.55.8:9000@19000 master - 0 1552425771407 13 connected 10923-16383
	8ed63224e33b12308587d6e999a43df69d7ffc24 10.211.55.7:8000@18000 master - 0 1552425773521 12 connected 5461-10922
	
可以看出 新增加的7002端口没有被分配solt

**为新实例分配slot**

redis 的 solt 是固定的，总共16384个，为新实例分配solt 需要将其他节点的solt分配到 7002实例上

下面例子是将100个solt分配到7002实例：

	$ redis-trib.rb reshard 10.211.55.6:7002
	>>> Performing Cluster Check (using node 10.211.55.6:7002)
	M: 0081c3db1996613d5cbb6b15910e4653b398a6fa 10.211.55.6:7002
	   slots: (0 slots) master
	   0 additional replica(s)
	S: b1ea02819986fabc3a722201c2d2ddcfcc0916f5 10.211.55.6:7001
	   slots: (0 slots) slave
	   replicates 8ed63224e33b12308587d6e999a43df69d7ffc24
	M: b7aab7e0ce75ff1c87c8b7cb0b0d2320becbe5b3 10.211.55.7:8001
	   slots:0-5460 (5461 slots) master
	   1 additional replica(s)
	S: d542d12e66bc2e5eec5d1eff04cc6afd31125a55 10.211.55.6:7000
	   slots: (0 slots) slave
	   replicates b7aab7e0ce75ff1c87c8b7cb0b0d2320becbe5b3
	S: 71a79ea31183031040e6527c6605a52ded3e95b0 10.211.55.8:9001
	   slots: (0 slots) slave
	   replicates 893974f5a56a3ff732e31dcece3a2b37fce421ea
	M: 893974f5a56a3ff732e31dcece3a2b37fce421ea 10.211.55.8:9000
	   slots:10923-16383 (5461 slots) master
	   1 additional replica(s)
	M: 8ed63224e33b12308587d6e999a43df69d7ffc24 10.211.55.7:8000
	   slots:5461-10922 (5462 slots) master
	   1 additional replica(s)
	[OK] All nodes agree about slots configuration.
	>>> Check for open slots...
	>>> Check slots coverage...
	[OK] All 16384 slots covered.
	How many slots do you want to move (from 1 to 16384)? 100
	What is the receiving node ID? 0081c3db1996613d5cbb6b15910e4653b398a6fa
	Please enter all the source node IDs.
	  Type 'all' to use all the nodes as source nodes for the hash slots.
	  Type 'done' once you entered all the source nodes IDs.
	Source node #1:all
	Ready to move 100 slots.
	  Source nodes:
	    M: b7aab7e0ce75ff1c87c8b7cb0b0d2320becbe5b3 10.211.55.7:8001
	   slots:0-5460 (5461 slots) master
	   1 additional replica(s)
	    M: 893974f5a56a3ff732e31dcece3a2b37fce421ea 10.211.55.8:9000
	   slots:10923-16383 (5461 slots) master
	   1 additional replica(s)
	    M: 8ed63224e33b12308587d6e999a43df69d7ffc24 10.211.55.7:8000
	   slots:5461-10922 (5462 slots) master
	   1 additional replica(s)
	  Destination node:
	    M: 0081c3db1996613d5cbb6b15910e4653b398a6fa 10.211.55.6:7002
	   slots: (0 slots) master
	   0 additional replica(s)
	  Resharding plan:
	  。。。
	  。。。
	  Do you want to proceed with the proposed reshard plan (yes/no)? yes
	  。。。

**将 7002实例 变成7000实例的slav**

由于7002上面有solt，所以转换失败

	$ redis-cli -c -h 10.211.55.6 -p 7002 cluster replicate d542d12e66bc2e5eec5d1eff04cc6afd31125a55
	(error) ERR To set a master the node must be empty and without assigned slots.
	
**删除 7002 节点**

由于7002节点已经分配solt，所以删除失败

	$ redis-trib.rb del-node 10.211.55.6:7002 0081c3db1996613d5cbb6b15910e4653b398a6fa 
	>>> Removing node 0081c3db1996613d5cbb6b15910e4653b398a6fa from cluster 10.211.55.6:7002
	[ERR] Node 10.211.55.6:7002 is not empty! Reshard data away and try again.

**将 7002节点 solt 分配到 7000节点**

	$ redis-trib.rb reshard 10.211.55.6:7002                                                 
	>>> Performing Cluster Check (using node 10.211.55.6:7002)
	M: 0081c3db1996613d5cbb6b15910e4653b398a6fa 10.211.55.6:7002
	   slots:0-32,5461-5494,10923-10955 (100 slots) master
	   0 additional replica(s)
	S: b1ea02819986fabc3a722201c2d2ddcfcc0916f5 10.211.55.6:7001
	   slots: (0 slots) slave
	   replicates 8ed63224e33b12308587d6e999a43df69d7ffc24
	S: b7aab7e0ce75ff1c87c8b7cb0b0d2320becbe5b3 10.211.55.7:8001
	   slots: (0 slots) slave
	   replicates d542d12e66bc2e5eec5d1eff04cc6afd31125a55
	M: d542d12e66bc2e5eec5d1eff04cc6afd31125a55 10.211.55.6:7000
	   slots:33-5460,5495-5543,10956-11005 (5527 slots) master
	   1 additional replica(s)
	S: 71a79ea31183031040e6527c6605a52ded3e95b0 10.211.55.8:9001
	   slots: (0 slots) slave
	   replicates 893974f5a56a3ff732e31dcece3a2b37fce421ea
	M: 893974f5a56a3ff732e31dcece3a2b37fce421ea 10.211.55.8:9000
	   slots:11006-16383 (5378 slots) master
	   1 additional replica(s)
	M: 8ed63224e33b12308587d6e999a43df69d7ffc24 10.211.55.7:8000
	   slots:5544-10922 (5379 slots) master
	   1 additional replica(s)
	[OK] All nodes agree about slots configuration.
	>>> Check for open slots...
	>>> Check slots coverage...
	[OK] All 16384 slots covered.
	How many slots do you want to move (from 1 to 16384)? 100
	What is the receiving node ID? d542d12e66bc2e5eec5d1eff04cc6afd31125a55
	Please enter all the source node IDs.
	  Type 'all' to use all the nodes as source nodes for the hash slots.
	  Type 'done' once you entered all the source nodes IDs.
	Source node #1:0081c3db1996613d5cbb6b15910e4653b398a6fa
	Source node #2:done

**再次查询，发现7002节点已经没有solt了**

	$ redis-cli -c -h 10.211.55.6 -p 7000 cluster nodes
	893974f5a56a3ff732e31dcece3a2b37fce421ea 10.211.55.8:9000@19000 master - 0 1552426926669 13 connected 11006-16383
	b1ea02819986fabc3a722201c2d2ddcfcc0916f5 10.211.55.6:7001@17001 slave 8ed63224e33b12308587d6e999a43df69d7ffc24 0 1552426926164 12 connected
	71a79ea31183031040e6527c6605a52ded3e95b0 10.211.55.8:9001@19001 slave 893974f5a56a3ff732e31dcece3a2b37fce421ea 0 1552426926567 13 connected
	0081c3db1996613d5cbb6b15910e4653b398a6fa 10.211.55.6:7002@17002 master - 0 1552426926668 17 connected
	8ed63224e33b12308587d6e999a43df69d7ffc24 10.211.55.7:8000@18000 master - 0 1552426927173 12 connected 5544-10922
	b7aab7e0ce75ff1c87c8b7cb0b0d2320becbe5b3 10.211.55.7:8001@18001 slave d542d12e66bc2e5eec5d1eff04cc6afd31125a55 0 1552426927678 18 connected
	d542d12e66bc2e5eec5d1eff04cc6afd31125a55 10.211.55.6:7000@17000 myself,master - 0 1552426925000 18 connected 0-5543 10923-11005

**再次删除 7002节点 成功，并且关闭7002节点实例**

	$ redis-trib.rb del-node 10.211.55.6:7002 0081c3db1996613d5cbb6b15910e4653b398a6fa
	>>> Removing node 0081c3db1996613d5cbb6b15910e4653b398a6fa from cluster 10.211.55.6:7002
	>>> Sending CLUSTER FORGET messages to the cluster...
	>>> SHUTDOWN the node.
	
**再次查询，发现已经没有7002节点了**

	$ redis-cli -c -h 10.211.55.6 -p 7000 cluster nodes                               
	893974f5a56a3ff732e31dcece3a2b37fce421ea 10.211.55.8:9000@19000 master - 0 1552427087077 13 connected 11006-16383
	b1ea02819986fabc3a722201c2d2ddcfcc0916f5 10.211.55.6:7001@17001 slave 8ed63224e33b12308587d6e999a43df69d7ffc24 0 1552427086572 12 connected
	71a79ea31183031040e6527c6605a52ded3e95b0 10.211.55.8:9001@19001 slave 893974f5a56a3ff732e31dcece3a2b37fce421ea 0 1552427088587 13 connected
	8ed63224e33b12308587d6e999a43df69d7ffc24 10.211.55.7:8000@18000 master - 0 1552427088000 12 connected 5544-10922
	b7aab7e0ce75ff1c87c8b7cb0b0d2320becbe5b3 10.211.55.7:8001@18001 slave d542d12e66bc2e5eec5d1eff04cc6afd31125a55 0 1552427088084 18 connected
	d542d12e66bc2e5eec5d1eff04cc6afd31125a55 10.211.55.6:7000@17000 myself,master - 0 1552427085000 18 connected 0-5543 10923-11005

**手动启动7002实例，再次查看7002实例`cluster nodes`信息**

检查7000端口，发现7002已经不再集群中了，再次检查7002端口，发现还能查看集训信息

说明，在删除节点的时候，只是在原有的集群中删除7002节点信息，但是7002节点自身的信息并没有删除，依然保存全部信息。

	$ redis-cli -c -h 10.211.55.6 -p 7002 cluster nodes
	8ed63224e33b12308587d6e999a43df69d7ffc24 10.211.55.7:8000@18000 master - 0 1552427282324 12 connected 5544-10922
	d542d12e66bc2e5eec5d1eff04cc6afd31125a55 10.211.55.6:7000@17000 master - 0 1552427281821 18 connected 0-5543 10923-11005
	b1ea02819986fabc3a722201c2d2ddcfcc0916f5 10.211.55.6:7001@17001 slave 8ed63224e33b12308587d6e999a43df69d7ffc24 0 1552427282324 12 connected
	71a79ea31183031040e6527c6605a52ded3e95b0 10.211.55.8:9001@19001 slave 893974f5a56a3ff732e31dcece3a2b37fce421ea 0 1552427281620 13 connected
	0081c3db1996613d5cbb6b15910e4653b398a6fa 10.211.55.6:7002@17002 myself,master - 0 1552427258203 17 connected
	893974f5a56a3ff732e31dcece3a2b37fce421ea 10.211.55.8:9000@19000 master - 0 1552427283334 13 connected 11006-16383
	b7aab7e0ce75ff1c87c8b7cb0b0d2320becbe5b3 10.211.55.7:8001@18001 slave d542d12e66bc2e5eec5d1eff04cc6afd31125a55 0 1552427282828 18 connected

而7002的全部信息，是记录在自身目录的nodes.conf中

	$ cat rdb/nodes_7002.conf 
	b1ea02819986fabc3a722201c2d2ddcfcc0916f5 10.211.55.6:7001@17001 slave 8ed63224e33b12308587d6e999a43df69d7ffc24 0 1552426921321 12 connected
	0081c3db1996613d5cbb6b15910e4653b398a6fa 10.211.55.6:7002@17002 myself,master - 0 1552426919000 17 connected
	b7aab7e0ce75ff1c87c8b7cb0b0d2320becbe5b3 10.211.55.7:8001@18001 slave d542d12e66bc2e5eec5d1eff04cc6afd31125a55 0 1552426921000 18 connected
	d542d12e66bc2e5eec5d1eff04cc6afd31125a55 10.211.55.6:7000@17000 master - 0 1552426920312 18 connected 0-5543 10923-11005
	71a79ea31183031040e6527c6605a52ded3e95b0 10.211.55.8:9001@19001 slave 893974f5a56a3ff732e31dcece3a2b37fce421ea 0 1552426921522 13 connected
	893974f5a56a3ff732e31dcece3a2b37fce421ea 10.211.55.8:9000@19000 master - 0 1552426921322 13 connected 11006-16383
	8ed63224e33b12308587d6e999a43df69d7ffc24 10.211.55.7:8000@18000 master - 0 1552426920111 12 connected 5544-10922
	vars currentEpoch 18 lastVoteEpoch 18
	
到其他节点查看，已经没有7002信息

	$ cat rdb/nodes_7000.conf 
	893974f5a56a3ff732e31dcece3a2b37fce421ea 10.211.55.8:9000@19000 master - 0 1552427057837 13 connected 11006-16383
	b1ea02819986fabc3a722201c2d2ddcfcc0916f5 10.211.55.6:7001@17001 slave 8ed63224e33b12308587d6e999a43df69d7ffc24 0 1552427055818 12 connected
	71a79ea31183031040e6527c6605a52ded3e95b0 10.211.55.8:9001@19001 slave 893974f5a56a3ff732e31dcece3a2b37fce421ea 0 1552427056000 13 connected
	8ed63224e33b12308587d6e999a43df69d7ffc24 10.211.55.7:8000@18000 master - 0 1552427056828 12 connected 5544-10922
	b7aab7e0ce75ff1c87c8b7cb0b0d2320becbe5b3 10.211.55.7:8001@18001 slave d542d12e66bc2e5eec5d1eff04cc6afd31125a55 0 1552427056021 18 connected
	d542d12e66bc2e5eec5d1eff04cc6afd31125a55 10.211.55.6:7000@17000 myself,master - 0 1552427055000 18 connected 0-5543 10923-11005
	vars currentEpoch 18 lastVoteEpoch 13


**再次将7002加入集群**

直接添加节点会报错，因为7002节点已经有数据了，需要清空节点数据再次添加

	$ redis-trib.rb add-node 10.211.55.6:7002 10.211.55.6:7000 
	>>> Adding node 10.211.55.6:7002 to cluster 10.211.55.6:7000
	>>> Performing Cluster Check (using node 10.211.55.6:7000)
	M: d542d12e66bc2e5eec5d1eff04cc6afd31125a55 10.211.55.6:7000
	   slots:0-5543,10923-11005 (5627 slots) master
	   1 additional replica(s)
	M: 893974f5a56a3ff732e31dcece3a2b37fce421ea 10.211.55.8:9000
	   slots:11006-16383 (5378 slots) master
	   1 additional replica(s)
	S: b1ea02819986fabc3a722201c2d2ddcfcc0916f5 10.211.55.6:7001
	   slots: (0 slots) slave
	   replicates 8ed63224e33b12308587d6e999a43df69d7ffc24
	S: 71a79ea31183031040e6527c6605a52ded3e95b0 10.211.55.8:9001
	   slots: (0 slots) slave
	   replicates 893974f5a56a3ff732e31dcece3a2b37fce421ea
	M: 8ed63224e33b12308587d6e999a43df69d7ffc24 10.211.55.7:8000
	   slots:5544-10922 (5379 slots) master
	   1 additional replica(s)
	S: b7aab7e0ce75ff1c87c8b7cb0b0d2320becbe5b3 10.211.55.7:8001
	   slots: (0 slots) slave
	   replicates d542d12e66bc2e5eec5d1eff04cc6afd31125a55
	[OK] All nodes agree about slots configuration.
	>>> Check for open slots...
	>>> Check slots coverage...
	[OK] All 16384 slots covered.
	[ERR] Node 10.211.55.6:7002 is not empty. Either the node already knows other nodes (check with CLUSTER NODES) or contains some key in database 0.


清空7002节点数据、cluster 信息，再次加入集群，成功

	$ redis-cli -h 10.211.55.6 -p 7002 flushall
	OK

	$ redis-cli -h 10.211.55.6 -p 7002 cluster reset
	OK
	
	$ redis-trib.rb add-node 10.211.55.6:7002 10.211.55.6:7000
	>>> Adding node 10.211.55.6:7002 to cluster 10.211.55.6:7000
	>>> Performing Cluster Check (using node 10.211.55.6:7000)
	M: d542d12e66bc2e5eec5d1eff04cc6afd31125a55 10.211.55.6:7000
	   slots:0-5543,10923-11005 (5627 slots) master
	   1 additional replica(s)
	M: 893974f5a56a3ff732e31dcece3a2b37fce421ea 10.211.55.8:9000
	   slots:11006-16383 (5378 slots) master
	   1 additional replica(s)
	S: b1ea02819986fabc3a722201c2d2ddcfcc0916f5 10.211.55.6:7001
	   slots: (0 slots) slave
	   replicates 8ed63224e33b12308587d6e999a43df69d7ffc24
	S: 71a79ea31183031040e6527c6605a52ded3e95b0 10.211.55.8:9001
	   slots: (0 slots) slave
	   replicates 893974f5a56a3ff732e31dcece3a2b37fce421ea
	M: 8ed63224e33b12308587d6e999a43df69d7ffc24 10.211.55.7:8000
	   slots:5544-10922 (5379 slots) master
	   1 additional replica(s)
	S: b7aab7e0ce75ff1c87c8b7cb0b0d2320becbe5b3 10.211.55.7:8001
	   slots: (0 slots) slave
	   replicates d542d12e66bc2e5eec5d1eff04cc6afd31125a55
	[OK] All nodes agree about slots configuration.
	>>> Check for open slots...
	>>> Check slots coverage...
	[OK] All 16384 slots covered.
	>>> Send CLUSTER MEET to node 10.211.55.6:7002 to make it join the cluster.
	[OK] New node added correctly.

**添加8002节点到集群中**

	$ redis-trib.rb add-node 10.211.55.7:8002 10.211.55.6:7000
	>>> Adding node 10.211.55.7:8002 to cluster 10.211.55.6:7000
	>>> Performing Cluster Check (using node 10.211.55.6:7000)
	M: d542d12e66bc2e5eec5d1eff04cc6afd31125a55 10.211.55.6:7000
	   slots:0-5543,10923-11005 (5627 slots) master
	   1 additional replica(s)
	M: 893974f5a56a3ff732e31dcece3a2b37fce421ea 10.211.55.8:9000
	   slots:11006-16383 (5378 slots) master
	   1 additional replica(s)
	S: b1ea02819986fabc3a722201c2d2ddcfcc0916f5 10.211.55.6:7001
	   slots: (0 slots) slave
	   replicates 8ed63224e33b12308587d6e999a43df69d7ffc24
	S: 71a79ea31183031040e6527c6605a52ded3e95b0 10.211.55.8:9001
	   slots: (0 slots) slave
	   replicates 893974f5a56a3ff732e31dcece3a2b37fce421ea
	M: 0081c3db1996613d5cbb6b15910e4653b398a6fa 10.211.55.6:7002
	   slots: (0 slots) master
	   0 additional replica(s)
	M: 8ed63224e33b12308587d6e999a43df69d7ffc24 10.211.55.7:8000
	   slots:5544-10922 (5379 slots) master
	   1 additional replica(s)
	S: b7aab7e0ce75ff1c87c8b7cb0b0d2320becbe5b3 10.211.55.7:8001
	   slots: (0 slots) slave
	   replicates d542d12e66bc2e5eec5d1eff04cc6afd31125a55
	[OK] All nodes agree about slots configuration.
	>>> Check for open slots...
	>>> Check slots coverage...
	[OK] All 16384 slots covered.
	>>> Send CLUSTER MEET to node 10.211.55.7:8002 to make it join the cluster.
	[OK] New node added correctly.
	
查看集群，现在是两个空的master节点，7002，8002

	$ redis-cli -c -h 10.211.55.6 -p 7002 cluster nodes       
	8ed63224e33b12308587d6e999a43df69d7ffc24 10.211.55.7:8000@18000 master - 0 1552428318549 12 connected 5544-10922
	d542d12e66bc2e5eec5d1eff04cc6afd31125a55 10.211.55.6:7000@17000 master - 0 1552428318549 18 connected 0-5543 10923-11005
	b1ea02819986fabc3a722201c2d2ddcfcc0916f5 10.211.55.6:7001@17001 slave 8ed63224e33b12308587d6e999a43df69d7ffc24 0 1552428317037 12 connected
	71a79ea31183031040e6527c6605a52ded3e95b0 10.211.55.8:9001@19001 slave 893974f5a56a3ff732e31dcece3a2b37fce421ea 0 1552428318045 13 connected
	0081c3db1996613d5cbb6b15910e4653b398a6fa 10.211.55.6:7002@17002 myself,master - 0 1552428317000 17 connected
	bf85776c71e5c37e43418ba25d963782d1ce6944 10.211.55.7:8002@18002 master - 0 1552428316534 0 connected
	893974f5a56a3ff732e31dcece3a2b37fce421ea 10.211.55.8:9000@19000 master - 0 1552428317037 13 connected 11006-16383
	b7aab7e0ce75ff1c87c8b7cb0b0d2320becbe5b3 10.211.55.7:8001@18001 slave d542d12e66bc2e5eec5d1eff04cc6afd31125a55 0 1552428318550 18 connected

**给7002分配solt**

	$ redis-trib.rb reshard 10.211.55.6:7000
	>>> Performing Cluster Check (using node 10.211.55.6:7000)
	M: d542d12e66bc2e5eec5d1eff04cc6afd31125a55 10.211.55.6:7000
	   slots:300-5543,10923-11005 (5327 slots) master
	   1 additional replica(s)
	M: bf85776c71e5c37e43418ba25d963782d1ce6944 10.211.55.7:8002
	   slots: (0 slots) master
	   0 additional replica(s)
	M: 893974f5a56a3ff732e31dcece3a2b37fce421ea 10.211.55.8:9000
	   slots:11006-16383 (5378 slots) master
	   1 additional replica(s)
	S: b1ea02819986fabc3a722201c2d2ddcfcc0916f5 10.211.55.6:7001
	   slots: (0 slots) slave
	   replicates 8ed63224e33b12308587d6e999a43df69d7ffc24
	S: 71a79ea31183031040e6527c6605a52ded3e95b0 10.211.55.8:9001
	   slots: (0 slots) slave
	   replicates 893974f5a56a3ff732e31dcece3a2b37fce421ea
	M: 0081c3db1996613d5cbb6b15910e4653b398a6fa 10.211.55.6:7002
	   slots:0-299 (300 slots) master
	   0 additional replica(s)
	M: 8ed63224e33b12308587d6e999a43df69d7ffc24 10.211.55.7:8000
	   slots:5544-10922 (5379 slots) master
	   1 additional replica(s)
	S: b7aab7e0ce75ff1c87c8b7cb0b0d2320becbe5b3 10.211.55.7:8001
	   slots: (0 slots) slave
	   replicates d542d12e66bc2e5eec5d1eff04cc6afd31125a55
	[OK] All nodes agree about slots configuration.
	>>> Check for open slots...
	>>> Check slots coverage...
	[OK] All 16384 slots covered.
	How many slots do you want to move (from 1 to 16384)? 300
	What is the receiving node ID? 0081c3db1996613d5cbb6b15910e4653b398a6fa
	Please enter all the source node IDs.
	  Type 'all' to use all the nodes as source nodes for the hash slots.
	  Type 'done' once you entered all the source nodes IDs.
	Source node #1:d542d12e66bc2e5eec5d1eff04cc6afd31125a55
	Source node #2:done

查看节点状态，发现7002已经从7000节点分配了300个solt

	$ redis-cli -c -h 10.211.55.6 -p 7002 cluster nodes
	8ed63224e33b12308587d6e999a43df69d7ffc24 10.211.55.7:8000@18000 master - 0 1552428524000 12 connected 5544-10922
	d542d12e66bc2e5eec5d1eff04cc6afd31125a55 10.211.55.6:7000@17000 master - 0 1552428524551 18 connected 300-5543 10923-11005
	b1ea02819986fabc3a722201c2d2ddcfcc0916f5 10.211.55.6:7001@17001 slave 8ed63224e33b12308587d6e999a43df69d7ffc24 0 1552428524650 12 connected
	71a79ea31183031040e6527c6605a52ded3e95b0 10.211.55.8:9001@19001 slave 893974f5a56a3ff732e31dcece3a2b37fce421ea 0 1552428524651 13 connected
	0081c3db1996613d5cbb6b15910e4653b398a6fa 10.211.55.6:7002@17002 myself,master - 0 1552428523000 19 connected 0-299
	bf85776c71e5c37e43418ba25d963782d1ce6944 10.211.55.7:8002@18002 master - 0 1552428523544 0 connected
	893974f5a56a3ff732e31dcece3a2b37fce421ea 10.211.55.8:9000@19000 master - 0 1552428523000 13 connected 11006-16383
	b7aab7e0ce75ff1c87c8b7cb0b0d2320becbe5b3 10.211.55.7:8001@18001 slave d542d12e66bc2e5eec5d1eff04cc6afd31125a55 0 1552428523000 18 connected

**将8002节点改为7002的slave**

	$ redis-cli -c -h 10.211.55.7 -p 8002 cluster replicate 0081c3db1996613d5cbb6b15910e4653b398a6fa     
	OK

查看状态,发现8002已经成为7002的slave

	$ redis-cli -c -h 10.211.55.6 -p 7002 cluster nodes                                             
	8ed63224e33b12308587d6e999a43df69d7ffc24 10.211.55.7:8000@18000 master - 0 1552428731960 12 connected 5544-10922
	d542d12e66bc2e5eec5d1eff04cc6afd31125a55 10.211.55.6:7000@17000 master - 0 1552428731456 18 connected 300-5543 10923-11005
	b1ea02819986fabc3a722201c2d2ddcfcc0916f5 10.211.55.6:7001@17001 slave 8ed63224e33b12308587d6e999a43df69d7ffc24 0 1552428732969 12 connected
	71a79ea31183031040e6527c6605a52ded3e95b0 10.211.55.8:9001@19001 slave 893974f5a56a3ff732e31dcece3a2b37fce421ea 0 1552428732969 13 connected
	0081c3db1996613d5cbb6b15910e4653b398a6fa 10.211.55.6:7002@17002 myself,master - 0 1552428730000 19 connected 0-299
	bf85776c71e5c37e43418ba25d963782d1ce6944 10.211.55.7:8002@18002 slave 0081c3db1996613d5cbb6b15910e4653b398a6fa 0 1552428732062 19 connected
	893974f5a56a3ff732e31dcece3a2b37fce421ea 10.211.55.8:9000@19000 master - 0 1552428732464 13 connected 11006-16383
	b7aab7e0ce75ff1c87c8b7cb0b0d2320becbe5b3 10.211.55.7:8001@18001 slave d542d12e66bc2e5eec5d1eff04cc6afd31125a55 0 1552428732666 18 connected


### 故障模拟

**模拟一： 其中一个master 宕机 到 恢复过程中，查看slave是否正常切换，对外服务是否正常**

1. 使用 `ruby example.rb` 持续写入数据

		$ ruby example.rb 
		
2. 重启 10.211.55.7 主机，并查看集群状态

	重启前：

		$ redis-cli -c -h 10.211.55.6 -p 7000 cluster nodes
		bf85776c71e5c37e43418ba25d963782d1ce6944 10.211.55.7:8002@18002 slave 0081c3db1996613d5cbb6b15910e4653b398a6fa 0 1552429112971 19 connected
		893974f5a56a3ff732e31dcece3a2b37fce421ea 10.211.55.8:9000@19000 master - 0 1552429112467 13 connected 11006-16383
		b1ea02819986fabc3a722201c2d2ddcfcc0916f5 10.211.55.6:7001@17001 slave 8ed63224e33b12308587d6e999a43df69d7ffc24 0 1552429112000 12 connected
		71a79ea31183031040e6527c6605a52ded3e95b0 10.211.55.8:9001@19001 slave 893974f5a56a3ff732e31dcece3a2b37fce421ea 0 1552429111561 13 connected
		0081c3db1996613d5cbb6b15910e4653b398a6fa 10.211.55.6:7002@17002 master - 0 1552429111963 19 connected 0-299
		8ed63224e33b12308587d6e999a43df69d7ffc24 10.211.55.7:8000@18000 master - 0 1552429112000 12 connected 5544-10922
		b7aab7e0ce75ff1c87c8b7cb0b0d2320becbe5b3 10.211.55.7:8001@18001 slave d542d12e66bc2e5eec5d1eff04cc6afd31125a55 0 1552429111561 18 connected
		d542d12e66bc2e5eec5d1eff04cc6afd31125a55 10.211.55.6:7000@17000 myself,master - 0 1552429113000 18 connected 300-5543 10923-11005


	重启过程中，写入数据失败
	
		ruby example.rb
		1
		2
		...
		...
		2036
		error Too many Cluster redirections? (last error: MOVED 9815 10.211.55.7:8000)
		error Too many Cluster redirections? (last error: MOVED 6072 10.211.55.7:8000)
		2039
		2040
		error Too many Cluster redirections? (last error: MOVED 7942 10.211.55.7:8000)
		2042
		2043
		2044
		error CLUSTERDOWN The cluster is down
		error CLUSTERDOWN The cluster is down
		error CLUSTERDOWN The cluster is down
		error CLUSTERDOWN The cluster is down
		2049


	查看集群状态，发现宕机节点8000的slave节点7001已经从slave 改成 master,并且800x节点已经是fail状态

		bf85776c71e5c37e43418ba25d963782d1ce6944 10.211.55.7:8002@18002 slave,fail 0081c3db1996613d5cbb6b15910e4653b398a6fa 1552429126774 1552429125569 19 disconnected
		893974f5a56a3ff732e31dcece3a2b37fce421ea 10.211.55.8:9000@19000 master - 0 1552429189000 13 connected 11006-16383
		b1ea02819986fabc3a722201c2d2ddcfcc0916f5 10.211.55.6:7001@17001 master - 0 1552429189558 20 connected 5544-10922
		71a79ea31183031040e6527c6605a52ded3e95b0 10.211.55.8:9001@19001 slave 893974f5a56a3ff732e31dcece3a2b37fce421ea 0 1552429190000 13 connected
		0081c3db1996613d5cbb6b15910e4653b398a6fa 10.211.55.6:7002@17002 master - 0 1552429189659 19 connected 0-299
		8ed63224e33b12308587d6e999a43df69d7ffc24 10.211.55.7:8000@18000 master,fail - 1552429126775 1552429124560 12 disconnected
		b7aab7e0ce75ff1c87c8b7cb0b0d2320becbe5b3 10.211.55.7:8001@18001 slave,fail d542d12e66bc2e5eec5d1eff04cc6afd31125a55 1552429126775 1552429126574 18 disconnected
		d542d12e66bc2e5eec5d1eff04cc6afd31125a55 10.211.55.6:7000@17000 myself,master - 0 1552429189000 18 connected 300-5543 10923-11005
	
	启动后，节点状态改为正常

		$ redis-cli -c -h 10.211.55.6 -p 7000 cluster nodes
		bf85776c71e5c37e43418ba25d963782d1ce6944 10.211.55.7:8002@18002 slave 0081c3db1996613d5cbb6b15910e4653b398a6fa 0 1552429356525 19 connected
		893974f5a56a3ff732e31dcece3a2b37fce421ea 10.211.55.8:9000@19000 master - 0 1552429356727 13 connected 11006-16383
		b1ea02819986fabc3a722201c2d2ddcfcc0916f5 10.211.55.6:7001@17001 master - 0 1552429356525 20 connected 5544-10922
		71a79ea31183031040e6527c6605a52ded3e95b0 10.211.55.8:9001@19001 slave 893974f5a56a3ff732e31dcece3a2b37fce421ea 0 1552429355000 13 connected
		0081c3db1996613d5cbb6b15910e4653b398a6fa 10.211.55.6:7002@17002 master - 0 1552429355517 19 connected 0-299
		8ed63224e33b12308587d6e999a43df69d7ffc24 10.211.55.7:8000@18000 slave b1ea02819986fabc3a722201c2d2ddcfcc0916f5 0 1552429355016 20 connected
		b7aab7e0ce75ff1c87c8b7cb0b0d2320becbe5b3 10.211.55.7:8001@18001 slave d542d12e66bc2e5eec5d1eff04cc6afd31125a55 0 1552429356525 18 connected
		d542d12e66bc2e5eec5d1eff04cc6afd31125a55 10.211.55.6:7000@17000 myself,master - 0 1552429354000 18 connected 300-5543 10923-11005



	
