---
layout: post
title: "CentOS 7 KVM 总结(二） KVM管理"
date: 2018-03-19 17:55:45 +0800
category: kvm
tags: [kvm]
---
* content
{:toc}



## 1. 常用kvm命令

- 列出所有虚拟机

		$ virsh list --all
		 Id    Name                           State
		----------------------------------------------------
		 54    ubuntu_16_04                   running
		 55    ubuntu_14_04                   running
		 73    k8s_m1                         running
		 74    k8s_m3                         running
		 76    k8s_m2                         running
		 
- 显示虚拟机信息

		$ virsh dominfo k8s_m1 
		Id:             73
		Name:           k8s_m1
		UUID:           2470dd0c-7956-44df-9b68-b1db5c30d502
		OS Type:        hvm
		State:          running
		CPU(s):         4
		CPU time:       231.5s
		Max memory:     4194304 KiB
		Used memory:    2097152 KiB
		Persistent:     yes
		Autostart:      enable
		Managed save:   no
		Security model: none
		Security DOI:   0

- 显示虚拟机内存和cpu使用情况,类似top

		More help in virt-top(1) man page. Press any key to return.
		virt-top 12:51:57 - x86_64 8/8CPU 800MHz 16266MB
		11 domains, 5 active, 5 running, 0 sleeping, 0 paused, 6 inactive D:0 O:0 X:0
		CPU: 0.3%  Mem: 10240 MB (10240 MB by guests)
		
		   ID S RDRQ WRRQ RXBY TXBY %CPU %MEM    TIME   NAME                                                                                                   
		   76 R    0    0  180    0  0.1 12.0   3:38.32 k8s_m2
		   74 R    0    0  180    0  0.1 12.0   3:51.02 k8s_m3
		   55 R    0    0  180    0  0.1 12.0   1:53.30 ubuntu_14_04
		   73 R    0    0  180    0  0.1 12.0   3:57.13 k8s_m1
		   54 R    0    0  180    0  0.0 12.0   2:06.37 ubuntu_16_04

- 显示分区信息

		$ virt-df k8s_m1
		Filesystem                           1K-blocks       Used  Available  Use%
		k8s_m1:/dev/sda1                       1038336     115276     923060   12%
		k8s_m1:/dev/centos/root               36936068    1188372   35747696    4%

	或

		$ virsh domblklist k8s_m1
		Target     Source
		------------------------------------------------
		vda        /data/kvmdiskk8s_m1.img
		
- 关闭虚拟机

		$ virsh shutdown k8s_m1
		
- 启动虚拟机

		$ virsh start k8s_m1
		
- 设置虚拟机开机启动

		$ virsh autostart k8s_m1

- 关闭虚拟及自启

		$ virsh undefine kvm-1
		
- 删除虚拟机

		$ virsh undefine k8s_m1

- 通过控制窗口登录虚拟机

		$ virsh console kvm-1


## 2. 调整虚拟机硬盘

- 查看虚拟机磁盘情况

		$ virt-df k8s_m1
		Filesystem                           1K-blocks       Used  Available  Use%
		k8s_m1:/dev/sda1                       1038336     115276     923060   12%
		k8s_m1:/dev/centos/root               36936068    1188372   35747696    4%


- 使用qemu-img 创建格式为qcow2的磁盘 

		$ qemu-img create -f qcow2 k8s_m1_data.img 10g
- 添加到虚拟机

		$ virsh attach-disk k8s_m1 /data/kvmdisk/k8s_m1_data.img vdb --cache writeback --subdriver qcow2 

- 进入虚拟机

		$ virsh console k8s_m1

- 挂载磁盘

		$ mkfs.ext4 /dev/vdb

		$ mount /dev/vdb /data/

- 查看虚拟机磁盘情况,发现已经多出一块新建的磁盘

		$ virt-df k8s_m1
		Filesystem                           1K-blocks       Used  Available  Use%
		k8s_m1:/dev/sdb                       10190100      36888    9612540    1%
		k8s_m1:/dev/sda1                       1038336     115276     923060   12%
		k8s_m1:/dev/centos/root      

## 3. 热修改虚拟机配置

### 3.1. 修改内存

1. 查看当前内存

		$ virsh dominfo k8s_m1 |grep memory
		Max memory:     4194304 KiB
		Used memory:    2097152 KiB

2. 修改配置

		$ virsh setmem k8s_m1 4194304

3. 查看修改后内存
	
		$ virsh dominfo k8s_m1 |grep memory
		Max memory:     4194304 KiB
		Used memory:    4194304 KiB

### 3.2. 修改CPU

1. 查看当前CPU

		$ virsh dominfo k8s_m1|grep CPU
		CPU(s):         4
		CPU time:       242.7s

2. 修改CPU

		$ virsh setvcpus k8s_m1 6

3. 查看修改后的CPU

		$ virsh dominfo k8s_m1|grep CPU
		CPU(s):         6
		CPU time:       251.1s

## 4. 删除虚拟机
	
	vm_name="k8s_m1"
	#第一步，停掉虚拟机
	virsh shutdown $vm_name
	#第二步
	virsh destroy $vm_name
	#第三步
	virsh undefine $vm_name
	#第四部
	rm /data/${vm_name}.img  # 不建议删除硬盘


## 5. 快照

### 5.1. 创建快照

	$ virsh snapshot-create-as kvm-1 kvm-1-original

### 5.2. 查看快照

	$ virsh snapshot-list kvm-1
	 Name                 Creation Time             State
	------------------------------------------------------------
	 kvm-1-original      2018-03-19 22:41:08 -0400 running

### 5.3. 进入虚拟机，安装mysql

	# 进入虚拟机
	$ virsh console kvm-1
	
	# 在虚拟机中yum安装mysql
	
	$ yum -y install http://dev.mysql.com/get/mysql57-community-release-el7-10.noarch.rpm
	$ yum -y install mysql-community-server
	# 启动服务
	$ systemctl start  mysqld.service
	# 获取初始密码
	$ mysql_pass=$(awk '/temporary password/ {print $NF}' /var/log/mysqld.log)
	# 登录
	$ mysql -uroot -p${mysql_pass}
	# 修改默认密码
	mysql> ALTER USER 'root'@'localhost' IDENTIFIED BY 'Nu@2KLTpeap9..@GeRRa';
	Query OK, 0 rows affected (0.00 sec)

#### 5.4. 退出虚拟机，恢复快照

	$ virsh snapshot-revert kvm-1 kvm-1-original

	# 重启虚拟机
	
	$ virsh shutdown kvm-1
	$ virsh start kvm-1

再次登录发现mysql已经不存在了


## 6. 问题整理

1. raw格式无法创建快照
		
		$ virsh snapshot-create CentOs6.8
		error: unsupported configuration: internal snapshot for disk vda unsupported for storage type raw

	
	**解决方法：**
	
	- 转换 raw 为 qcow2
	
			qemu-img convert -f raw -O qcow2 /data/kvm/CentOs6.8.img /data/kvm/centos6.8.qcow2
	
	- 修改虚拟机配置文件

			virsh edit vm_name
			# 此命令编辑的文件实际上是/etc/libvirt/qemu/目录下和虚拟机同名并且以xml结尾的文件
			virsh edit CentOs6.8 

			#将type和source file修改为指定格式。
			
			<disk type='file' device='disk'>
			
			      <driver name='qemu' type='qcow2' cache='none'/>
			
			      <source file='/data/kvm/centos6.8.qcow2'/>
			
			      <target dev='vda' bus='virtio'/>
			
			      <address type='pci' domain='0x0000' bus='0x00' slot='0x05' function='0x0'/>
			
			    </disk>

	- 创建快照

			virsh snapshot-create CentOs6.8
			Domain snapshot 1457182425 created


			
	
	


