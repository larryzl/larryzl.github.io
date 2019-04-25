---
layout: post
title: "Linux 命令总结之 find"
date: 2016-12-25 14:17:36 +0800
category: Linux命令 
tags: [Linux命令]
---
* content
{:toc}

### 1. 介绍

> Linux find命令用来在指定目录下查找文件。任何位于参数之前的字符串都将被视为欲查找的目录名。如果使用该命令时，不设置任何参数，则find命令将在当前目录下查找子目录与文件。并且将查找到的子目录和文件全部进行显示。
> 

**语法：**

	find   path   -option   [   -print ]   [ -exec   -ok   command ]   {} \;
	
**参数说明 :**

> `find` 根据下列规则判断 `path` 和 `expression`，在命令列上第一个 - ( ) , ! 之前的部份为 path，之后的是 expression。如果 path 是空字串则使用目前路径，如果 expression 是空字串则使用 -print 为预设 expression。
> 
>expression 中可使用的选项有二三十个之多，在此只介绍最常用的部份。

|expression|解释|
|---|---|
|-mount, -xdev | 只检查和指定目录在同一个文件系统下的文件，避免列出其它文件系统中的文件|
|-amin n | 在过去 n 分钟内被读取过|
|-anewer file | 比文件 file 更晚被读取过的文件|
|-atime n | 在过去n天内被读取过的文件|
|-cmin n | 在过去 n 分钟内被修改过|
|-cnewer file |比文件 file 更新的文件|
|-ctime n|  在过去n天内被修改过的文件|
|-empty | 空的文件-gid n or -group name : gid 是 n 或是 group 名称是 name|
|-ipath p, -path p | 路径名称符合 p 的文件，ipath 会忽略大小写|
|-name name, -iname name | 文件名称符合 name 的文件。iname 会忽略大小写|
|-size n | 文件大小 是 n 单位，b 代表 512 位元组的区块，c 表示字元数，k 表示 kilo bytes，w 是二个位元组。|
|-type c | 文件类型是 c 的文件。|
|d| 目录|
|c| 字型装置文件|
|b| 区块装置文件|
|p| 具名贮列|
|f| 一般文件|
|l| 符号连结|
|s| socket|
|-pid n | process id 是 n 的文件|
|你可以使用 ( ) 将运算式分隔，并使用下列运算。|
|exp1 -and exp2|
|! expr|
|-not expr|
|exp1 -or exp2|
|exp1, exp2|

### 2. 实例

1. 将目前目录及其子目录下所有延伸档名是 c 的文件列出来。

		# find . -name '*.c'
		
2. 将目前目录其其下子目录中所有一般文件列出

		# find . -type f
		
3. 将目前目录及其子目录下所有最近 20 天内更新过的文件列出

		# find . -ctime -20
		
4. 查找/var/log目录中更改时间在7日以前的普通文件，并在删除之前询问它们：

		# find /var/log -type f -mtime +7 -ok rm {} \;

5. 查找前目录中文件属主具有读、写权限，并且文件所属组的用户和其他用户具有读权限的文件：

		# find . -type f -perm 644 -exec ls -l {} \;

6. 为了查找系统中所有文件长度为0的普通文件，并列出它们的完整路径：

		# find / -type f -size 0 -exec ls -l {} \;

7. 查找目录

		# find . -type d

8. 查找名字为test的目录

		# find . -name test

9. 查找名字符合正则表达式的文件，注意前面的`.*`（查找到的文件袋有目录）

		# find . -regex .*so.*\.gz

10. 查找目录并列出目录下的文件(为找到的每一个目录单独执行`ls`命令，没有选项`-print`时文件列表前一行不会显示目录名称）

		# find ./ -type d -print -exec ls {} \;

11. 查找目录并列出目录下的文件(为找到的每一个目录单独执行`ls`命令，执行命令前需要确认）

		# find ./ -type d -ok ls {} \;

12. 查找目录并列出目录下的文件（将找到的目录添加到`ls`命令后一次执行，参数过长时会分多次执行）

		# find ./ -type d -exec ls {} +

13. 查找文件名匹配`.c`的文件

		# find ./ -name \*.c

14. 打印test文件名后，打印test文件的内容

		# find ./ -name test -print -exec cat {} \;

15. 不打印test文件名，只打印test文件的内容

		# find ./ -name test -exec cat {} \;

16. 查找文件更新日时在距现在时刻二天以内的文件

		# find ./ -mtime -2

17. 查找文件更新日时在距现在时刻二天以上的文件

		# find ./ -mtime +2

18. 查找文件更新日时在距离现在时刻一天以上二天以内的文件

		# find ./ -mtime 2

19. 查找文件更新日时在距现在时刻2分以内的文件

		# find ./ -mmin -2

20. 查找文件更新日时在距现在时刻2分以上的文件

		# find ./ -mmin +2

21. 查找文件更新时间比文件abc的内容更新时间新的文件

		# find ./ -newer abc

22. 查找文件访问时间比文件abc的内容更新时间新的文件

		# find ./ -anewer abc

23. 查找空文件或空目录

		# find ./ -empty

24. 查找空文件并删除

		# find ./ -empty -type f -print -delete

25. 查找权限为644的文件或目录(需完全符合)

		# find ./ -perm 664

26. 查找用户/组权限为读写，其他用户权限为读(其他权限不限)的文件或目录

		# find ./ -perm -664

27. 查找用户有写权限或者组用户有写权限的文件或目录

		# find ./ -perm /220
		# find ./ -perm /u+w,g+w
		# find ./ -perm /u=w,g=w

28. 查找所有者权限有读权限的目录或文件

		# find ./ -perm -u=r

29. 查找用户组权限有读权限的目录或文件

		# find ./ -perm -g=r

30. 查找其它用户权限有读权限的目录或文件

		# find ./ -perm -o=r

31. 查找所有者为lzj的文件或目录

		# find ./ -user lzj

32. 查找组名为gname的文件或目录

		# find ./ -group gname

33. 查找文件的用户ID不存在的文件

		# find ./ -nouser

34. 查找文件的组ID不存在的文件

		# find ./ -nogroup

35. 查找有执行权限但没有可读权限的文件

		# find ./ -executable \! -readable

36. 查找文件size小于10个字节的文件或目录

		# find ./ -size -10c

37. 查找文件size等于10个字节的文件或目录

		# find ./ -size 10c

38. 查找文件size大于10个字节的文件或目录

		# find ./ -size +10c

39. 查找文件size小于10k的文件或目录

		# find ./ -size -10k

40. 查找文件size小于10M的文件或目录

		# find ./ -size -10M

41. 查找文件size小于10G的文件或目录

		# find ./ -size -10G


#3. find错误总结

1. 解决find命令报错： `paths must precede expression`

		find /tmp  -maxdepth 1 -mtime 30 -name *.pdf 
		find: paths must precede expression
		Usage: find [-H] [-L] [-P] [path...] [expression
		
	出现`paths must precede expression` 错误原因： 多文件的查找的时候需要增加单引号。修改为：

		find ./ -mtime +30 -type f -name '*.php'


