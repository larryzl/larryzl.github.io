---
layout: post
title: "Linux 命令总结之 vmstat"
date: 2016-03-07 17:24:23 +0800
category: Linux命令
tags: [Linux命令]
---
* content
{:toc}

## 简介

free 命令是一个显示系统中空闲和已用内存大小的工具。free 命令的输出和 top 命令相似。大多数Linux发行版已经含有 free 命令。

## 如何运行 free

想要运行，只需在控制台输入free 即可。不带选项运行会显示一个以KB为单位的默认输出。

	$ free
	             total       used       free     shared    buffers     cached
	Mem:        107104     102648       4456        180       6664      50376
	-/+ buffers/cache:      45608      61496
	Swap:       524284          0     524284

从上面的截图我们看到：

**内存 (以KB计)**

- Total（全部） : 107104
- Used（已用） : 102648
- Free（可用） : 4456
- Shared（共享） : 180
- Buffers（块设备缓存区） : 6664
- Cached（文件缓存） : 50376

buffers是指用来给块设备做的缓冲大小，他只记录文件系统的metadata以及 tracking in-flight pages.

cached是用来给文件做缓冲。

那就是说：buffers是用来存储，目录里面有什么内容，权限等等。而cached直接用来记忆我们打开的文件

**Swap (以KB计)**

- Total（全部） : 524284
- Used（已用） : 0
- Free（可用） : 524284

当你看见 buffer/cache 的空闲空间低或者 swap 的空闲空间低，说明内存需要升级了。这意味这内存利用率很高。请注意 shared（共享）内存列应该被忽略 ，因为它已经被废弃了。

## 以其它单元显示内存信息

如我们先前提到的，默认 free 会以 KB 为单位显示信息。free 同样提供给我们 b (B), -k (KB), -m (MB), -g (GB) and –tera (TB)这些单位。要显示我们想要的单位，只要选择一个并在 free 后面跟上。下面一个是以 MB 为单位的输出样例。

	$ free -m
	             total       used       free     shared    buffers     cached
	Mem:           104        100          4          0          6         49
	-/+ buffers/cache:         44         60
	Swap:          511          0        511

这个技巧同样适用于-b, -k, -g 以及 –tera 选项。

## 显示高低内存利用率

如果我们想要知道高低内存统计，我们可以使用-l选项。下面是一个例子。

	$ free -l
	             total       used       free     shared    buffers     cached
	Mem:        107104     102648       4456        180       6672      50376
	Low:        107104     102648       4456
	High:            0          0          0
	-/+ buffers/cache:      45600      61504
	Swap:       524284          0     524284

## 显示 Linux 全部内存

如果我们需要每列的总计信息，我们可以在 free 命令后面跟上 -t 选项。这会在字底部额外加入一行显示。

	$ free -t
	             total       used       free     shared    buffers     cached
	Mem:        107104     102648       4456        180       6672      50376
	-/+ buffers/cache:      45600      61504
	Swap:       524284          0     524284
	Total:      631388     102648     528740

## 总结

除了vmstat以外，free 命令也是一个用于统计内存利用率的简单统计工具。用这个你可以快速查看你的 Linux 内存信息。free 命令使用 /proc/meminfo 作为基准来显示内存利用率信息。如往常一样，你可以在控制台下输入 man free 来获取更多关于 free 的信息。

