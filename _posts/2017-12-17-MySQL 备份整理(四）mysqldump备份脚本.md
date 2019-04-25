---
layout: post
title: "MySQL 备份整理(四）mysqldump备份脚本"
date: 2017-12-21 14:23:29 +0800
category: MySQL
tags: [MySQL]
---
* content
{:toc}


`mysqldump` 备份脚本

指定 脚本中的变量放入`crontab` 中定期执行

	#!/bin/bash
	###########
	# - Data: 2017-12
	# - Version: 0.5
	# - Author: larryzl
	# - Desc: Mysql mysqldump 定时备份脚本； 备份策略  每周几进行全备，其他时间进行增量备份
	###########
	
	###################################自定义变量开始######################################
	# - 定义变量
	# - MySQL 连接信息
	MYSQL_USER="root"
	MYSQL_PASS=""
	MYSQL_PORT=3306
	MYSQL_HOST="localhost"
	# - MySQL 目录信息
	MYSQL_BASE_DIR=/Data/apps/mysql
	MYSQL_DATA_DIR=${MYSQL_BASE_DIR}/data
	MYSQL_BIN_DIR=${MYSQL_BASE_DIR}/bin
	MYSQL_SOCKET=${MYSQL_BASE_DIR}/mysql3306.sock    
	MYSQL_BIN_INDEX=${MYSQL_DATA_DIR}/mysql-bin.index
	# - 备份目录信息
	MYSQL_BACKUP_DIR=/bak                       # - 备份目录
	MYSQL_BACKUP_LOG=/bak/mysql_backup.log      # - 备份日志
	MYSQL_BACKUP_OVERDUE=30                     # - 备份保留时间
	
	# - 定义每周备份日期 1-7 代表每周几进行全备份
	full_backup_date=1
	
	###################################自定义变量结束######################################
	
	# - 配置环境变量
	export PATH=${PATH}:${MYSQL_BIN_DIR}
	
	# - 时间标记
	TODAY_TIME_TAG=$(date +"%Y-%m-%d")          # - 当前日期
	OVERDUE_TIME_TAG=$(date --date="${MYSQL_BACKUP_OVERDUE} days ago" +"%Y-%m-%d")  # - 过期日期
	
	# - 定义备份名称
	myisam_dump_file=${MYSQL_BACKUP_DIR}/${TODAY_TIME_TAG}/MyISAM-${TODAY_TIME_TAG}-full.sql.gz   # - 当前myisam引擎备份文件名称
	innodb_dump_file=${MYSQL_BACKUP_DIR}/${TODAY_TIME_TAG}/InnoDB-${TODAY_TIME_TAG}-full.sql.gz   # - 当前innodb引擎备份文件名称
	old_file_myisam=${MYSQL_BACKUP_DIR}/${OVERDUE_TIME_TAG}/MyISAM-${OVERDUE_TIME_TAG}-full.sql.gz  # - 过期myisam引擎备份文件名称
	old_file_innodb=${MYSQL_BACKUP_DIR}/${OVERDUE_TIME_TAG}/InnoDB-${OVERDUE_TIME_TAG}-full.sql.gz  # - 过期innodb引擎备份文件名称
	
	# - 连接数据库字段
	MYSQL_CONN="-h${MYSQL_HOST} -u${MYSQL_USER} -p${MYSQL_PASS} -S${MYSQL_SOCKET} -P${MYSQL_PORT}"
	
	# - 需要备份的所有数据库名称
	backup_dbs=$(mysql $MYSQL_CONN -e "show databases;" |egrep -v "Database|information_schema|mysql|performance_schema"|tr '\n' ' ')
	myisam_db="mysql"
	
	# - 发送通知方法
	function send_notify(){
	    echo "send notify"
	}
	
	# - 打印备份结果
	function bakcup_resault(){
	    if [ $? -eq 0 ];then
	        msg="[$(date +"%Y-%m-%d %H:%M:%S")] Full backup $1 successfully."
	    else
	        msg="[$(date +"%Y-%m-%d %H:%M:%S")] Full backup $1 failed."
	        # - 发送失败通知
	        # send_notify $msg      
	    fi
	    # - 打印日志
	    echo ${msg} >> ${MYSQL_BACKUP_LOG}
	        
	}
	
	# - 删除过期文件
	function delete_overdue_files(){
	    if [ -f $1 ];then
	        rm -rf $1
	        msg="[$(date +"%Y-%m-%d %H:%M:%S")] Delete overdue file $1 successfully." 
	    else
	        msg="[$(date +"%Y-%m-%d %H:%M:%S")] Delete overdue file $1 failed, $1 not exist. "
	    fi
	    # - 打印日志
	    echo ${msg} >> ${MYSQL_BACKUP_LOG}
	}
	
	# - 备份binary log
	function backup_binary_log(){
	    # - 当前binary log日志
	    CURRENT_BINARY_LOG=$(cat ${MYSQL_BIN_INDEX} |sed -n -e 's/\.\///g' -e '$p')
	    # - 当前binary log前缀
	    BINARY_LOG_INDEX=$(echo ${CURRENT_BINARY_LOG} |awk -F '.' '{print $1}')
	    # - 需要备份的binary log列表
	    BACKUP_BINARY_LOG=$(cat ${MYSQL_BIN_INDEX} | sed -e '$d' -e 's/\.\///g')
	    # - 备份目录
	    BINARY_LOG_BACKUP_DIR=${MYSQL_BACKUP_DIR}/$(date +"%Y-%m-%d")/
	    # - 开始拷贝
	    for binary_log_file in ${BACKUP_BINARY_LOG}
	    do
	        cp ${MYSQL_DATA_DIR}/${binary_log_file} ${BINARY_LOG_BACKUP_DIR}
	        echo "[$(date +"%Y-%m-%d %H:%M:%S")] binary log ${binary_log_file} backup to ${BINARY_LOG_BACKUP_DIR} ok" >> ${MYSQL_BACKUP_LOG}
	    done
	    # - 删除一天前的binary log
	    mysql ${MYSQL_CONN} -e "purge master logs to '${CURRENT_BINARY_LOG}'"
	    echo "[$(date +"%Y-%m-%d %H:%M:%S")] purge master logs to '${CURRENT_BINARY_LOG}'">> ${MYSQL_BACKUP_LOG} 
	}
	
	# - 全备方法
	function full_all_backup(){
	
	    echo "[$(date +"%Y-%m-%d %H:%M:%S")] mysql full backup start." >> ${MYSQL_BACKUP_LOG}
	    
	    INNODB_ARGS="--single-transaction  --master-data=2 --events --triggers --routines --flush-logs --databases ${backup_dbs}"
	
	    MYISAM_ARGS="--lock-all-tables --flush-privileges --databases  ${myisam_db}"
	
	    # - 备份Innodb数据库
	    ${MYSQL_BIN_DIR}/mysqldump ${MYSQL_CONN} ${INNODB_ARGS}  | gzip > ${innodb_dump_file} 2>>${MYSQL_BACKUP_LOG}
	    bakcup_resault $innodb_dump_file "InnoDB"
	    # - Innodb备份完以后刷新新的mysql-bin日志，将之前的binary log文件备份到备份目录,然后删除
	    
	    # - 备份binary log
	    backup_binary_log
	
	    # - 备份MyisAM数据库
	    ${MYSQL_BIN_DIR}/mysqldump ${MYSQL_CONN} ${MYISAM_ARGS} | gzip > ${myisam_dump_file} 2>>${MYSQL_BACKUP_LOG}
	    bakcup_resault $myisam_dump_file "MyISAM"
	
	    echo "[$(date +"%Y-%m-%d %H:%M:%S")] mysql full backup stop." >> ${MYSQL_BACKUP_LOG}
	
	}
	
	# - 初始化增量备份方法
	function init_increment_backup(){
	    # - 增量备份只需要备份相应的binary log即可
	    echo "[$(date +"%Y-%m-%d %H:%M:%S")] mysql increment backup start." >> ${MYSQL_BACKUP_LOG}
	    # 刷新binary log
	    ${MYSQL_BIN_DIR}/mysqladmin ${MYSQL_CONN} flush-logs
	    echo "[$(date +"%Y-%m-%d %H:%M:%S")] mysqladmin flush logs ok." >> ${MYSQL_BACKUP_LOG}
	    backup_binary_log
	    echo "[$(date +"%Y-%m-%d %H:%M:%S")] mysql increment backup stop." >> ${MYSQL_BACKUP_LOG}
	    echo "" >> ${MYSQL_BACKUP_LOG}
	}
	
	# - 初始化全备方法
	function init_full_backup(){
	
	    full_all_backup
	
	    delete_overdue_files ${old_file_myisam}
	    delete_overdue_files ${old_file_innodb}
	
	    echo "" >> ${MYSQL_BACKUP_LOG}
	}
	
	# 执行备份程序
	function run(){
	    # - 创建备份目录
	    [ ! -d ${MYSQL_BACKUP_DIR} ] && mkdir -p ${MYSQL_BACKUP_DIR}
	
	    # - 创建备份日志文件
	    [ ! -f ${MYSQL_BACKUP_LOG} ] && touch ${MYSQL_BACKUP_LOG}
	    # - 创建备份目录
	    [ ! -f ${MYSQL_BACKUP_DIR}/$(date +"%Y-%m-%d") ] && mkdir -p ${MYSQL_BACKUP_DIR}/$(date +"%Y-%m-%d")
	
	    week_of_day=$(date +"%u")
	    # - 定义默认值
	    : ${full_backup_date:=7}
	
	    if [ ${week_of_day} == ${full_backup_date} ];then
	        init_full_backup
	    else
	        init_increment_backup
	    fi
	}
	
	run