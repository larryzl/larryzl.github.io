---
layout: post
title: "go程序托管 supervisor 遇到 too many open files"
date: 2019-03-26 10:26:45 +0800
category: Linux
tags: [Linux]
---
* content
{:toc}



使用golang写了一个统计日志到程序，最开始使用 `nohup` 来运行，一切正常，后来该换到使用 supervisor 来管理进程，运行一天后，日志出现大量的 `too many open files` 错误，开始排查...

首先排查系统文件描述符

```
$ cat /etc/security/limits.conf

* soft nofile 655350
* hard nofile 655350
```



查看 日志进程 的`limit`

```
$cat /proc/<pid>/limits
Limit                     Soft Limit           Hard Limit           Units
Max cpu time              unlimited            unlimited            seconds
Max file size             unlimited            unlimited            bytes
Max data size             unlimited            unlimited            bytes
Max stack size            8388608              unlimited            bytes
Max core file size        0                    unlimited            bytes
Max resident set          unlimited            unlimited            bytes
Max processes             127897               127897               processes
Max open files            1024                2048                files
Max locked memory         65536                65536                bytes
Max address space         unlimited            unlimited            bytes
Max file locks            unlimited            unlimited            locks
Max pending signals       127897               127897               signals
Max msgqueue size         819200               819200               bytes
Max nice priority         0                    0
Max realtime priority     0                    0
Max realtime timeout      unlimited            unlimited            us

```

发现系统设置的正常，但是到程序中就只有默认的 1024，开始排查 supervisor

```
$ cat /proc/$(ps aux|grep supervisor|grep -v grep |awk '{print $2}')/limits
Limit                     Soft Limit           Hard Limit           Units
Max cpu time              unlimited            unlimited            seconds
Max file size             unlimited            unlimited            bytes
Max data size             unlimited            unlimited            bytes
Max stack size            8388608              unlimited            bytes
Max core file size        0                    unlimited            bytes
Max resident set          unlimited            unlimited            bytes
Max processes             127897               127897               processes
Max open files            1024                2048                files
Max locked memory         65536                65536                bytes
Max address space         unlimited            unlimited            bytes
Max file locks            unlimited            unlimited            locks
Max pending signals       127897               127897               signals
Max msgqueue size         819200               819200               bytes
Max nice priority         0                    0
Max realtime priority     0                    0
Max realtime timeout      unlimited            unlimited            us
```

发现 supervisor进程 也是 1024, 修改supervisor 配置 `/etc/supervisord.conf`


调整参数 `minfds=655350`， 然后重启supervisor

