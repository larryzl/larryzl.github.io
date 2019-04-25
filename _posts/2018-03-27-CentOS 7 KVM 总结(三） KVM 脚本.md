---
layout: post
title: "CentOS 7 KVM 总结(三） KVM 脚本"
date: 2018-03-27 01:59:42 +0800
category: kvm
tags: [kvm]
---
* content
{:toc}

KVM 脚本

	#!/usr/bin/env python3
	
	import yaml
	import os
	import sys
	import logging
	
	
	logging.basicConfig(level=logging.INFO,format = '%(asctime)s - %(name)s - %(levelname)s - %(message)s')
	logger = logging.getLogger(__name__)
	
	class Manger(object):
	    def __init__(self):
	        with open('./Model.yaml') as f:
	            self.ModelConfig = yaml.safe_load(f)
	
	
	    def __isVmExist(self,**kwargs):
	        vm_name = kwargs.get('vm_name',None)
	
	
	        if vm_name:
	            try:
	                vm_name_list = os.popen("virsh list --all").read().split()[4:]
	                if len(vm_name_list)<0:
	                    return True
	                else:
	                    s = 1
	                    for n in range(s,len(vm_name_list),2):
	                        if vm_name_list[n] == vm_name:
	                            return False
	                    else:
	                        return True
	            except:
	                raise("获取主机名称错误")
	
	    def __choiceSpec(self):
	        '''
	        选择 主机模板
	        :return:
	        '''
	        num = 1
	        spec_list = []
	        print("---------------------------------------------------------------------------------")
	        for i in self.ModelConfig:
	            if i['type'] == 'spec':
	
	                print("[%2s] %s" % (num,i['name']))
	                print("CPU: %3s \t Max CPU: %3s ||" % (i['cpus']['vcpus'],i['cpus']['maxvcpus']),end="\t")
	                print("Mem: %3s \t Max Mem: %3s ||" % (i['memory']['memory'],i['memory']['maxmemory']),end="\t")
	                print("Disk: %3s G" % (i['disk']))
	                print("OS : %20s ||" % (i['os-type']),end="\t")
	                print("Variant: %3s" % (i['os-variant']))
	                print("---------------------------------------------------------------------------------")
	                num+=1
	                spec_list.append(i)
	        if len(spec_list) == 0:
	            logger.error("没有找到虚拟机配置模板,请检查")
	            sys.exit(1)
	        while 1:
	            c = input("选择虚拟机配置模板编号:")
	            try:
	                c = int(c)
	                if 0 < c <= num:
	                    return spec_list[c-1]
	                else:
	                    logger.error("配置模板编号输入错误")
	
	            except:
	                logger.error("配置模板编号输入错误,模板不存在")
	                continue
	
	    def __choiceItem(self,type):
	        '''
	        选择存储目录/镜像源/连接方式
	        :param type:
	        :return:
	        '''
	        num = 1
	
	        for i in self.ModelConfig:
	            if i['type'] == type:
	                item_list = i
	                break
	        for i in item_list['options']:
	            print("[%2s] %s" % (item_list['options'].index(i),i))
	        while True:
	            try:
	                c = input('choice %s:' % item_list['name'])
	                if c == "quit":
	                    sys.exit(1)
	                c = int(c)
	                return item_list['options'][c]
	            except:
	                logger.info("编号输入错误")
	                continue
	
	
	
	
	    def createKvm(self):
	        logger.info("开始创建KVM虚拟机")
	        while 1:
	            vm_name = input("virtual machine name:")
	            if len(vm_name) > 5:break
	            print("主机名过短,重新输入")
	        if not self.__isVmExist(vm_name=vm_name):
	            logger.info("已存在名为 %s 的虚拟机" % vm_name)
	
	        spec = self.__choiceSpec()
	        print(spec)
	        iso = self.__choiceItem('iso')
	        storage = self.__choiceItem('storage')
	        connect = self.__choiceItem('connect')
	
	        cmd = "virt-install --name=%s --memory=%s,maxmemory=%s --vcpus=%s,maxvcpus=%s " \
	              "--os-type=%s --os-variant=%s --disk path=%s%s.img,size=%s " \
	            "--bridge=br0 --autostart " % (vm_name,spec['memory']['memory'],spec['memory']['maxmemory'],
	                                              spec['cpus']['vcpus'],spec['cpus']['maxvcpus'],spec['os-type'],
	                                              spec['os-variant'],storage,vm_name,spec['disk'])
	
	        if connect == "console":
	            args = '--location=%s --graphics=none --console=pty,target_type=serial --extra-args="console=tty0 console=ttyS0"' % iso
	        elif connect == "vnc":
	            import random
	            while 1:
	                vnc_port = random.randint(5900,6000)
	                try:
	                    r = os.popen("ss -ntlp |grep %s|grep -v grep" % vnc_port).read()
	                    if len(r) == 0:
	                        break
	                    else:
	                        continue
	                except:
	                    continue
	
	            args = "--cdrom=%s --vnc --vnclisten=0.0.0.0 --vncport=%s -d" % (iso,vnc_port)
	
	        cmd = cmd+args
	        print(cmd)
	        os.system(cmd)
	
	
	
	if __name__ == "__main__":
	    m = Manger()
	    m.createKvm()


脚本配置文件

	- type: spec
	  name: small
	  memory:
	    memory: 256
	    maxmemory: 1024
	  cpus:
	    vcpus: 2
	    maxvcpus: 4
	  disk: 10
	  os-type: linux
	  os-variant: rhel7
	
	- type: spec
	  name: middle
	  memory:
	    memory: 512
	    maxmemory: 1024
	  cpus:
	    vcpus: 2
	    maxvcpus: 4
	  disk: 20
	  os-type: linux
	  os-variant: rhel7
	
	- type: spec
	  name: large
	  memory:
	    memory: 1024
	    maxmemory: 2048
	  cpus:
	    vcpus: 2
	    maxvcpus: 4
	  disk: 40
	  os-type: linux
	  os-variant: rhel7
	
	- type: iso
	  name: "image source"
	  options:
	    - "/data/iso/CentOS-7-x86_64-Minimal-1511.iso"
	
	- type: storage
	  name: "Storage directory"
	  options:
	    - "/data/kvmdisk"
	
	- type: connect
	  name: "Connection mode"
	  options:
	    - vnc
	    - console
	
	- type: global
	  debug: true
	
	
