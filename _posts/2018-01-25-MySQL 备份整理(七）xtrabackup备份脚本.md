---
layout: post
title: "MySQL 备份整理(七）xtrabackup备份脚本."
date: 2018-01-25 15:02:17 +0800
category: MySQL
tags: [MySQL]
---
* content
{:toc}

> xtrabackup 备份脚本

	#!/bin/bash
	#set -e
	#
	# Desc: Xtrabackup MySQL备份脚本
	# CreateDate: 2017-02-19 11:27:17
	# LastModify: 
	# Author: larryzl
	# 
	# Note:
	#   MySql 自动备份脚本  
	#
	# History:
	#
	#
	# ---------------------- Script Begin ----------------------
	#
	
	
	# ---------------------- 定义全局变量 ----------------------
	#- 全备日期 day of week (0..6); 0 is Sunday
	FULL_BACKUP_DATE=5
	#- 定义备份保存周期,删除前N个周期的备份
	SAVE_CYCLE=2
	#- 用户名密码信息
	MYSQL_USER="xtrabackup"
	MYSQL_PASSWORD='12345678'
	#- 存储目录
	BACKUP_DIR="/Data/xtrabackup"
	#- 备份数据库名,多个数据库用双引号括起来,留空表示全部数据库
	DATABASE="flow company eshop flow_yizai flow_bak ott_auto ott_data pingan pt website zentao"
	#- 备份日志
	LOG="/home/mysql/scripts/auto_xtrabackup.log"
	#- 是否启用压缩备份
	IS_COMPRESS=true
	#- 压缩线程数
	COMPRESS_THREADS=2
	#- 拷贝线程数
	PARALLEL_THREADS=4
	### MY_CNF 与 DATADIR 指定一个即可
	#- my.cnf位置 
	#MY_CNF="/etc/my.cnf"
	#- 数据库数据目录
	DATADIR="/Data/apps/mysql/data"
	#- 命令位置
	XTRABACKUP="/usr/local/xtrabackup/bin/xtrabackup"
	#- 记录最近一次全备目录名称文件
	last_backup_lock_file="${BACKUP_DIR}/lastbackup_info"
	#- 记录备份列表
	backup_list="${BACKUP_DIR}/backup_list"
	
	
	
	# ---------------------- 函数开始 ----------------------
	
	today=$(date +"%Y-%m-%d_%H-%M-%S")
	
	#- xtrabackup 执行程序
	function xtrabackupExec(){
	    ${XTRABACKUP} --user=${MYSQL_USER} --password=${MYSQL_PASSWORD} $*    
	}
	
	#- 记录日志
	function logSave(){
	    t=$(date +"[%Y-%m-%d %H:%M:%S]")
	    echo "${t} $*" >> ${LOG}
	}
	
	#- 全备方法
	function fullBackup(){
	    #- 备份存储目录
	    target_dir="${BACKUP_DIR}/fullbackup/${today}"
	    #- 记录xtrabackup备份过程日志
	    log_file="${BACKUP_DIR}/fullbackup/${today}.log"
	
	    #- 打印日志
	    logSave "-------------------------"
	    logSave "正在进行全量备份..."
	    logSave "数据库:  $([ ! -z "${DATABASE}" ] && echo ${DATABASE} || echo 全库)"
	    logSave "MySQL数据目录:  ${DATADIR}"
	    logSave "xtrabackup日志:   ${log_file}"
	    logSave "备份存储目录:   ${target_dir}"
	    logSave "是否启用压缩:   ${IS_COMPRESS}"
	
	    [ -d ${target_dir} ] || mkdir -p ${target_dir}
	    ARGS="--backup --datadir=${DATADIR}"
	    #- 判断是否启用压缩
	    if [ ${IS_COMPRESS} == true ];then 
	        ARGS="${ARGS} --parallel=${PARALLEL_THREADS} --compress --compress-threads=${COMPRESS_THREADS}"
	    else
	        ARGS="${ARGS} --parallel=${PARALLEL_THREADS}"
	    fi
	    
	
	    ARGS="${ARGS}  --target-dir=${target_dir}"
	
	    begin_time=$(date +%s)
	    logSave "开始备份。。。"
	    if [ "${DATABASE}0" == "0" ];then
	        xtrabackupExec ${ARGS} > ${log_file} 2>&1 
	    else    
	        ${XTRABACKUP} --user=${MYSQL_USER} --password=${MYSQL_PASSWORD} ${ARGS} --databases="${DATABASE}" > ${log_file} 2>&1 
	    fi
	    if [ $? -ne 0 ];then
	        logSave "备份出错，请查看xtrabackup日志，${log_file}"
	        exit 1
	    fi
	    end_time=$(date +%s)
	    #- 备份完成后记录当前备份目录
	    echo "${target_dir}" > $last_backup_lock_file
	    echo "${target_dir}" >> $backup_list
	    logSave "备份完成，共用时:$((${end_time} - ${begin_time} )) 秒"
	    exit 0
	}
	#- 增备方法
	function incrementalBackup(){
	    #- 备份目录
	    target_dir="${BACKUP_DIR}/incremental/${today}"
	    #- xtrabackup执行日志
	    log_file="${BACKUP_DIR}/incremental/${today}.log"
	    
	    #- 打印日志
	    logSave "-------------------------"
	    logSave "正在进行增量备份..."
	    logSave "数据库:  $([ ! -z "${DATABASE}" ] && echo ${DATABASE} || echo 全库)"
	    logSave "MySQL数据目录:  ${DATADIR}"
	    logSave "xtrabackup日志:   ${log_file}"
	    logSave "备份存储目录:   ${target_dir}"
	
	    [ -d ${target_dir} ] || mkdir -p ${target_dir}
	
	    ARGS="--backup --parallel=${PARALLEL_THREADS} --datadir=${DATADIR}"
	    #- 判断是否备份全部数据库
	    
	
	    if [ ! -f ${last_backup_lock_file} ];then
	        logSave "未找到最近一次全备目录，请检查！"
	        exit 1
	    fi
	
	    incremental_basedir=`cat $last_backup_lock_file`
	    ARGS="${ARGS} --target-dir=${target_dir} --incremental-basedir=${incremental_basedir}"
	
	    begin_time=$(date +%s)
	    logSave "开始备份。。。"
	    if [ ! -z "${DATABASE}" ];then
	        ${XTRABACKUP} --user=${MYSQL_USER} --password=${MYSQL_PASSWORD} ${ARGS} --databases="${DATABASE}" > ${log_file} 2>&1 
	    else
	        xtrabackupExec ${ARGS} > ${log_file} 2>&1 
	    fi
	    if [ $? -ne 0 ];then
	        logSave "备份出错，请查看xtrabackup日志，${log_file}"
	        exit 1
	    fi
	    end_time=$(date +%s)
	    #- 备份完成后记录当前备份目录
	    echo "${target_dir}" > $last_backup_lock_file
	    echo "${target_dir}" >> $backup_list
	    logSave "备份完成，共用时:$((${end_time} - ${begin_time} )) 秒"
	    exit 0
	
	}
	
	#- 还原方法
	#
	# 还原方法需手动执行 
	#
	function copyBack(){
	    #- 是否为增量还原  true,false
	    is_incremental=$1
	    fullbackup_path=$2
	    incremental_path=$3
	
	    if [ "${is_incremental}0" == "true0" ];then
	        if [ $# -ne 3 ];then
	            echo "参数错误"
	            exit 1
	        fi
	        ${XTRABACKUP} --prepare --apply-log-only --target-dir=${fullbackup_path} --incremental-dir=${incremental_path}
	    else
	        if [ $# -ne 2 ];then
	            echo "参数错误"
	            exit 1
	        fi
	        ${XTRABACKUP} --prepare --apply-log-only --target-dir=${fullbackup_path}
	    fi
	}
	
	#- 删除备份
	function delBackup(){
	    #- 只在全备日期删除 备份周期以前的日志
	    if [ $(date +"%w") != ${FULL_BACKUP_DATE} ];then
	        exit 0
	    fi
	
	    delDays=$((($SAVE_CYCLE + 1) * 7))
	    while [ $delDays -gt $(($SAVE_CYCLE * 7)) ]
	    do
	        for del_file in $(grep $(date --date="${delDays} days ago" +"%Y-%m-%d_%H") $backup_list)
	        do 
	            if [ -d ${del_file} ];then
	                logSave "过期日志删除成功，日志目录名: ${del_file}"
	                rm -rf $del_file
	            else
	                logSave "过期日志删除失败，目录不存在，日志目录名: ${del_file}"
	            fi
	            sed -i "/${del_file}/d" $backup_list
	        done
	        delDays=$((${delDays} - 1))
	    done
	
	
	}
	
	function Usage(){
	    echo "`basename $0` Missing argument"
	    echo "USAGE:`basename $0` -a | -f | -i"
	    echo "-a    auto backup"
	    exit 1
	}
	
	
	
	# warning
	if [[ $# -lt 1 ]];then
	    echo -n "正在手动执行脚本，前请确保参数配置正确(y/n)"
	    read r
	    : ${r=:'n'}
	    [ ${r} != 'y' ] && $(echo "exit script" ;exit)
	    echo "注意，手动操作可能导致数据错误，请谨慎执行!"
	    echo "f) 进行全量备份"
	    echo "i) 进行增量备份"
	    echo "c) 进行还原"
	    echo -n "请输入要进行的操作: "
	    read f
	    if [ "${f}0" == "0" ];then
	        echo "输入错误，退出。"
	        exit 1
	    elif [ "${f}0" == "f0" ];then
	        echo -n "选择全量备份，请确认(y/n):"
	        read c
	        if [ "${c}0" == "y0" ];then
	            fullBackup
	        else
	            echo "输入错误，退出。"
	            exit 1
	        fi
	    elif [ "${f}0" == "i0" ];then
	        echo -n "选择增量备份，请确认(y/n):"
	        read c
	        if [ "${c}0" == "y0" ];then
	            incrementalBackup
	        else
	            echo "输入错误，退出。"
	            exit 1
	        fi
	    elif [[ "${f}0" == "c0" ]]; then
	        echo -n "是否进行数据还原，请确认(y/n):"
	        read c 
	        if [ "${c}0" != "y0" ];then
	            echo "输入错误，退出。"
	        fi
	        echo "进入数据还原（输入q退出操作，回车则使用缺省值）"
	        echo -n "1) 请输入需要prepare的全备路径($(grep fullbackup ${backup_list}|sed -n '$p')): "
	        read fullbackup_path
	        fullbackup_path=$(grep fullbackup ${backup_list}|sed -n '$p')
	        if [[ "${fullbackup_path}0" == "0" ]];then
	            echo "未找到最近一次全备目录，请检查！"    
	            exit 1
	        fi
	        echo "正在对${fullbackup_path} 进行 prepare ..."
	        copyBack "false" ${fullbackup_path}
	        num=1
	        while [[ 1 -eq 1 ]]; do
	            echo -n "是否继续prepare增量备份(y/n)?:"
	            read c_prepare
	            if [ "${c_prepare}0" == "0" ];then
	                continue
	            elif [[ "${c_prepare}0" == "y0" ]]; then
	                prepare_path=$(grep -A ${num} fullbackup ${backup_list}|sed -n '$p'|grep -v "fullbackup")
	                
	                echo -n "2) 请输入需要prepare的增备路径(${prepare_path}): "
	                read path
	                if [ "${path}0" != "0" ];then
	                    prepare_path=$path
	                fi
	                echo "正在对 增量备份 ${prepare_path} 进行 prepare"
	                copyBack 'true' ${fullbackup_path} ${prepare_path}
	                num=$(( $num + 1 ))
	            else
	                copyBack "false" ${fullbackup_path}
	                echo "完成prepare"
	            fi  
	            #statements
	        done
	        
	        
	
	
	
	    else
	        echo "输入错误，退出。"
	        exit 1
	    fi
	fi
	
	while getopts :a OPTION;do
	    case $OPTION in
	    a)
	        date_w=$(date +"%w")
	        if [ ${date_w} ==  ${FULL_BACKUP_DATE} ];then
	            fullBackup
	        else
	            incrementalBackup
	        fi
	    ;;
	
	    *)
	       Usage
	    ;;
	    esac
	done




