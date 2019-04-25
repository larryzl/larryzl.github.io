---
layout: post
title: "Linux 命令总结之 crond.md"
date: 2016-03-08 09:31:13 +0800
category: Linux命令
tags: [Linux命令]
---
* content
{:toc}

## 简介

crond是linux下用来周期性的执行某种任务或等待处理某些事件的一个守护进程，与windows下的计划任务类似，当安装完成操作系统后，默认会安装此服务工具，并且会自动启动crond进程，crontab依赖的服务就是crond，crond进程每分钟会定期检查是否有要执行的任务，如果有要执行的任务，则自动执行该任务。这个crond定时任务服务就相当于我们生活中的闹钟！

由于crond 是Linux的内置服务，但它不自动起来，可以用以下的方法启动、关闭这个服务

	/sbin/service crond start //启动服务
	/sbin/service crond stop //关闭服务
	/sbin/service crond restart //重启服务
	/sbin/service crond reload //重新载入配置
	或者使用下面的命令：
	/etc/init.d/crond start
	/etc/init.d/crond restart
	/etc/init.d/crond stop

设置crond服务开机自启动：

	[root@gin tmp]# chkconfig crond on

特殊需要：crond服务搞不定了，一般工作中写脚本守护程序执行：

	[root@gin tmp]# cat cron.sh
	while true
	do
	        echo "I am Bruce Lee"
	        sleep 1
	done

程序文件：程序代码组成，但是没有在计算机内执行。当前没有执行。

守护程序或守护进程：进程就是所算机中正在执行的程序，守护进程就是保护正在一直运行的程序。

Linux下的任务调度分为两类，系统任务调度和用户任务调度。

**A. 系统任务调度：**

系统周期性所要执行的工作，比如写缓存数据到硬盘、日志清理等。在/etc目录下有一个crontab文件，这个就是系统任务调度的配置文件。

/etc/crontab文件包括下面几行：

	SHELL=/bin/bash 
	PATH=/sbin:/bin:/usr/sbin:/usr/bin 
	MAILTO=root 
	HOME=/ 
	# run-parts 
	01 * * * * root run-parts /etc/cron.hourly 
	02 4 * * * root run-parts /etc/cron.daily 
	22 4 * * 0 root run-parts /etc/cron.weekly 
	42 4 1 * * root run-parts /etc/cron.monthly

前四行是用来配置crond任务运行的环境变量

第一行SHELL变量指定了系统要使用哪个shell，这里是bash

第二行PATH变量指定了系统执 行命令的路径

第三行MAILTO变量指定了crond的任务执行信息将通过电子邮件发送给root用户，如果MAILTO变量的值为空，则表示不发送任 务执行信息给用户

第四行的HOME变量指定了在执行命令或者脚本时使用的主目录。第六至九行表示的含义将在下个小节详细讲述。这里不在多说。

**B. 用户任务调度：**

用户定期要执行的工作，比如用户数据备份、定时邮件提醒等。用户可以使用 crontab 工具来定制自己的计划任务。所有用户定义的crontab 文件都被保存在 /var/spool/cron目录中。其文件名与当前用户名一致。

严格的说，linux系统下的定时任务软件还真不少，例如：at , crontab , anacron。

at：适合仅执行一次就结束的调度任务命令，如：某天晚上需要处理一个任务，仅仅是这一天的晚上，属于突发性的工作任务。要执行at命令，还需要启动一个名为atd的服务才行。在生产环境中此需要会很少用到。因此，建议不要深入研究！

提示：

1. 我们所说的crond服务是运行的程序，而crontab命令用户用来设置定时规则的命令

2. crond服务是企业生产工作中常用的重要服务，at和anacron很少使用，可以忽略

3. 几乎每个服务器都会用到crond服务

## crontab工具的使用

1. crontab的使用格式

	crontab常用的使用格式有如下两种：

		crontab [-u user] [file]
		crontab [-u user] [-e|-l|-r |-i]
		
	选项含义如下：

	|参数名称|含义|指定示例|
	|---|---|---|
	|-l|显示用户crontab文件内容，l即list|crontab -l|
	|-e|编辑某个用户的crontab文件内容。如果不指定用户，则表示编辑当前用户的crontab文件。|crontab -e|
	|-i|在删除用户的crontab文件时给确认提示。|crontab -ri|
	|-r|从/var/spool/cron目录中删除某个用户的crontab文件，不指定用户，则删除当前用户的crontab文件。|crontab -r
	|-u|用来设定某个用户的crontab服务,此参数一般由root用户来运行|crontab -u root -l|
	
	常用的选项命令为`crontab -e` 相当于 `vi /var/spool/cron/root` ；` crontab -l `相当于 `cat /var/spool/cron/root`
	
	使用者权限文件：
	
	|文件|说明|
	|---|---|
	|/etc/cron.deny|该文件中所列用户不允许使用crontab命令|
	|/etc/cron.allow|该文件中所列用户允许使用crontab命令|
	|/var/spool/cron/|所有用户crontab文件存放的目录，以用户名命令|
	
2. crontab文件的含义

	用户所建立的crontab文件中，每一行都代表一项任务，每行的每个字段代表一项设置，它的格式共分为六个字段，前五段是时间设定段，第六段是要执行的命令段，格式如下：

		minute   hour   day   month   week   command
	
	其中：

		* minute： 表示分钟，可以是从0到59之间的任何整数。
		* hour：表示小时，可以是从0到23之间的任何整数。
		* day：表示日期，可以是从1到31之间的任何整数。
		* month：表示月份，可以是从1到12之间的任何整数。
		* week：表示星期几，可以是从0到7之间的任何整数，这里的0或7代表星期日。
		* command：要执行的命令，可以是系统命令，也可以是自己编写的脚本文件。

	在以上各个字段中，还可以使用以下特殊字符：

		* 星号（*）：代表所有可能的值，例如month字段如果是星号，则表示在满足其它字段的制约条件后每月都执行该命令操作。
		* 逗号（,）：可以用逗号隔开的值指定一个列表范围，例如，“1,2,5,7,8,9”
		* 中杠（-）：可以用整数之间的中杠表示一个整数范围，例如“2-6”表示“2,3,4,5,6”
		* 正斜线（/）：可以用正斜线指定时间的间隔频率，例如“0-23/2”表示每两小时执行一次。同时正斜线可以和星号一起使用，例如*/10，如果用在minute字段，表示每十分钟执行一次。

3. crontab文件举例

		# 表示每隔3个小时重启apache服务一次。 
		0 */3 * * * /usr/local/apache2/apachectl restart

		#表示每周六的3点30分执行/webdata/bin/backup.sh脚本的操作。  
		30 3 * * 6 /webdata/bin/backup.sh

		#表示每个月的1号和20号检查/dev/sdb8磁盘设备。
		0 0 1,20 * * fsck /dev/sdb8

		#表示每个月的5号、10号、15号、20号、25号、30号的5点10分执行清理apache日志操作。
		10 5 */5 * *  echo "">/usr/local/apache2/log/access_log

4. 任务调度设置文件的写法

	可用crontab -e命令来编辑,编辑的是/var/spool/cron下对应用户的cron文件,也可以直接修改/etc/crontab文件

	具体格式如下：

		Minute Hour Day Month Dayofweek   command
		分钟     小时   天     月       天每星期       命令
		每个字段代表的含义如下：
		Minute             每个小时的第几分钟执行该任务
		Hour               每天的第几个小时执行该任务
		Day                 每月的第几天执行该任务
		Month             每年的第几个月执行该任务
		DayOfWeek     每周的第几天执行该任务
		Command       指定要执行的程序


	在这些字段里，除了“Command”是每次都必须指定的字段以外，其它字段皆为可选字段，可视需要决定。对于不指定的字段，要用“*”来填补其位置。

	举例如下：

		5       *       *           *     *     ls             指定每小时的第5分钟执行一次ls命令
		30     5       *           *     *     ls             指定每天的 5:30 执行ls命令
		30     7       8         *     *     ls             指定每月8号的7：30分执行ls命令
		30     5       8         6     *     ls             指定每年的6月8日5：30执行ls命令
		30     6       *           *     0     ls             指定每星期日的6:30执行ls命令[注：0表示星期天，1表示星期1，

	以此类推，也可以用英文来表示，sun表示星期天，mon表示星期一等。

		30     3     10,20     *     *     ls     每月10号及20号的3：30执行ls命令[注：“，”用来连接多个不连续的时段
		25     8-11 *           *     *     ls       每天8-11点的第25分钟执行ls命令[注：“-”用来连接连续的时段
		*/15   *       *           *     *     ls         每15分钟执行一次ls命令 [即每个小时的第0 15 30 45 60分钟执行ls命令
		30   6     */10         *     *     ls       每个月中，每隔10天6:30执行一次ls命令[即每月的1、11、21、31日是的6：30执行一次ls 命令。

	每天7：50以root 身份执行/etc/cron.daily目录中的所有可执行文件


		50   7       *             *     *     root     run-parts     /etc/cron.daily   [ 注：run-parts参数表示，执行后面目录中的所有可执行文件。

5. cron文件语法

		分     小时    日       月       星期     命令
		0-59   0-23   1-31   1-12     0-6     command     (取值范围,0表示周日一般一行对应一个任务)

	记住几个特殊符号的含义:

		“*”代表取值范围内的数字,
		“/”代表”每”,
		“-”代表从某个数字到某个数字,
		“,”分开几个离散的数字

6. 应用案例
		
		# 在本例中，该定时任务的意思就是每天凌晨3点30分和中午12点30分执行/scripts/andy.sh
		30 3,12 * * * /bin/sh /scripts/andy.sh
		
		# 每隔6个小时的半点时刻执行后面的脚本任务
		30 */6 * * * /bin/sh /scripts/andy.sh

		# 8点到18点中每隔2小时的30分执行后面的脚本任务（也就是每天的8点半，10点半，12点半，14点半，16点半，18点半）
		30 8-18/2 * * * /bin/sh /scripts/andy.sh

		# 每月的1，10，22日的凌晨4：45分重启apache
		45 4 1,10,22 * * /application/apache/bin/apachectl graceful

##使用crontab工具的注意事项

1. 注意环境变量问题

	有时我们创建了一个crontab，但是这个任务却无法自动执行，而手动执行这个任务却没有问题，这种情况一般是由于在crontab文件中没有配置环境变量引起的。
	
	在crontab文件中定义多个调度任务时，需要特别注意的一个问题就是环境变量的设置，因为我们手动执行某个任务时，是在当前shell环境下进行的， 程序当然能找到环境变量，而系统自动执行任务调度时，是不会加载任何环境变量的，因此，就需要在crontab文件中指定任务运行所需的所有环境变量，这样，系统执行任务调度时就没有问题了。

2. 注意清理系统用户的邮件日志

	每条任务调度执行完毕，系统都会将任务输出信息通过电子邮件的形式发送给当前系统用户，这样日积月累，日志信息会非常大，可能会影响系统的正常运行，因此，将每条任务进行重定向处理非常重要。
	
	例如，可以在crontab文件中设置如下形式，忽略日志输出：

		0 */3 * * * /usr/local/apache2/apachectl restart >/dev/null 2>&1

	“/dev/null 2>&1”表示先将标准输出重定向到/dev/null，然后将标准错误重定向到标准输出，由于标准输出已经重定向到了/dev/null，因此标准错误也会重定向到/dev/null，这样日志输出问题就解决了。
	
	为crontab中的任务增加自己的日志，这样出错后，比较容易看到原因。
	
		0 6 * * * $HOME/for_crontab/createTomorrowTables >> $HOME/for_crontab/mylog.log 2>&1

	把错误输出和标准输出都输出到mylog.log中。

	注意：不要写成

		0 6 * * * $HOME/for_crontab/createTomorrowTables 2>&1 >> $HOME/for_crontab/mylog.log

	否则就输出到标准输出了

3. 系统级任务调度与用户级任务调度

	系统级任务调度主要完成系统的一些维护操作，用户级任务调度主要完成用户自定义的一些任务，可以将用户级任务调度放到系统级任务调度来完成（不建议这么做），但是反过来却不行，root用户的任务调度操作可以通过“crontab –uroot –e”来设置，也可以将调度任务直接写入/etc/crontab文件，需要注意的是，如果要定义一个定时重启系统的任务，就必须将任务放到/etc /crontab文件，即使在root用户下创建一个定时重启系统的任务也是无效的。

## 企业生产场景如何调试crontab定时任务

1. 增加执行任务频率调试任务（某些任务不能用于生产环境，没有测试机会）

	代码发布：个人开发环境 -- 办公测试环境 -- IDC机房测试环境 -- IDC正式环境（分组，灰度发布）

2. 调整系统时间调试任务（不能直接用于生产环境），保持五分钟

3. 通过脚本日志输出调试定时任务

4. 注意一些任务命令带来的问题

5. 注意环境变量导致的定时任务故障（java环境变量问题： http://oldboy.blog.51cto.com/2561410/1541515）

6. 通过crond定时任务服务日志调试定时任务（/var/log/cron）

7. 其他稀奇古怪的问题调试的办法

## crontab定时任务生产应用问题八箴言

1. 系统环境变量问题

2. 定时任务要用绝对路径

3. 脚本权限问题加/bin/sh

4. 时间变量问题用反斜线\%转义，最好用脚本

5. ：>/dev/null 2>&1问题（1>/dev/null  2>/dev/null  &>/dev/null）

6. 定时任务规则之前加注释

7. 使用脚本程序替代命令行定时任务

8. 避免不必要的程序及命令输出

9. 切到目标目录的上一级打包目标

10. 定时任务脚本中的程序命令尽量用全路径（和环境变量的识别有关）





		
		
		


