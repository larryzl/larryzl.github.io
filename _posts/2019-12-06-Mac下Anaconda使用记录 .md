---
layout: post
title: "Mac下Anaconda使用记录(转）"
date: 2019-12-06 12:11:27 +0800
category: python
tags: [python]
---
* content
{:toc}

- [anaconda](https://www.anaconda.com)
- [pycharm](https://www.jetbrains.com/pycharm/download/)

## 1. 下载Anaconda并安装

1. 下载安装脚本

	`wget https://repo.anaconda.com/archive/Anaconda3-2019.07-MacOSX-x86_64.sh`

2. 执行脚本

	`bash Anaconda3-2019.07-MacOSX-x86_64.sh`

3. 安装完成后，在`~/.zshrc`中添加

	```
	# >>> conda initialize >>>
	# !! Contents within this block are managed by 'conda init' !!
	__conda_setup="$('/Users/lei/anaconda3/bin/conda' 'shell.bash' 'hook' 2> /dev/null)"
	if [ $? -eq 0 ]; then
	    eval "$__conda_setup"
	else
	    if [ -f "/Users/lei/anaconda3/etc/profile.d/conda.sh" ]; then
	        . "/Users/lei/anaconda3/etc/profile.d/conda.sh"
	    else
	        export PATH="/Users/lei/anaconda3/bin:$PATH"
	    fi
	fi
	unset __conda_setup
	# <<< conda initialize <<<
	```

4. 刷新环境变量

	`source ~/.zshrc`
	
5. 查看是否有conda命令

		$ conda -V
		conda 4.7.10

## 2. conda基本命令

### 2.1. 环境管理命令

- 创建新的python环境：`$ conda create --name myenv`
- 并且还可以指定python的版本：`$ conda create -n myenv python=3.7`
- 创建新环境并指定包含的库：`$ conda create -n myenv scipy`
- 并且还可以指定库的版本：`$ conda create -n myenv scipy=0.15.0`
- 复制环境：`$ conda create --name myclone --clone myenv`
- 查看是不是复制成功了：`$ conda info --envs`
- 激活、进入某个环境：`$ source activate myenv`
- 退出环境：`$ conda deactivate / $ source deactivate`
- 删除环境：`$ conda remove --name myenv --all`
- 查看当前的环境列表$ `conda info --envs / $ conda env list`

### 2.2. 包/库管理命令

- 查看conda版本：`$ conda --version`
- 更新conda版本：`$ conda update conda / anaconda`
- 查看都安装了那些依赖库：`$ conda list`
- 更新所有库 `$ conda update --all`
- 查看某个环境下安装的库：`$ conda list -n myenv`
- 查找包：`$ conda search <package>`
- 安装包：`$ conda install <package>`
- 安装到指定环境：`$ conda install -n myenv <package>`
- 更新包：`$ conda update <package>`
- 删除包：`$ conda remove <package>`

## 3. 创建虚拟环境

### 3.1. 创建虚拟环境

命令： `$ conda create -n learn python=3.7`

- `-n learn` 指定环境名称为`learn`
- `python=3.7` 指定python版本


### 3.2. 查看`learn`环境是否创建成功

命令： `$ conda env list`

```
conda env list
# conda environments:
#
base                     /Users/xxx/anaconda3
learn                 *  /Users/xxx/anaconda3/envs/learn
pytorch                  /Users/xxx/anaconda3/envs/pytorch
```

显示当前`anacoda`共有3个虚拟环境，其中`base`是默认环境

### 3.3. 启动虚拟环境 `$ conda activate learn`

### 3.4. 关闭虚拟环境` $ conda deactivate`









本文转载自： [链接](https://www.jianshu.com/p/ce99bf9d9008)


