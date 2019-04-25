---
layout: post
title: "ElasticSearch 整理(二) 设置"
date: 2018-03-28 22:48:01 +0800
category: ElasticSearch
tags: [ElasticSearch,ELK]
---
* content
{:toc}


# 1. ElasticSearch 安装&目录结构

## 1.1. 安装

	wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-6.5.4.tar.gz
	wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-6.5.4.tar.gz.sha512
	shasum -a 512 -c elasticsearch-6.5.4.tar.gz.sha512 
	tar -xzf elasticsearch-6.5.4.tar.gz
	cd elasticsearch-6.5.4/

## 1.2. 自动创建`X-PACK`索引

X-Pack将尝试在Elasticsearch中自动创建多个索引。默认情况下，Elasticsearch配置为允许自动创建索引，不需要其他步骤。但是，如果你有Elasticsearch禁用自动创建索引，你必须配置 [action.auto_create_index](https://www.elastic.co/guide/en/elasticsearch/reference/6.5/docs-index_.html#index-creation)的elasticsearch.yml，让X-包创建以下指标：

	action.auto_create_index: .monitoring*,.watches,.triggered_watches,.watcher-history*,.ml*
		
## 1.3. 命令行启动

	./bin/elasticsearch

## 1.4. 守护进程运行
	
启动
	
	./bin/elasticsearch -d -p pid
	
关闭
	
	kill `cat pid`

## 1.5. 在命令行配置ElasticSearch

ES_HOME/config/elasticsearch.yml 默认情况下，Elasticsearch从文件加载其配置。配置Elasticsearch中介绍了此配置文件的格式 。
	
可以在命令行中指定可在配置文件中指定的任何设置，使用以下-E语法：
	
	./bin/elasticsearch -d -Ecluster.name=my_cluster -Enode.name=node_1

## 1.6. ElasticSearch 目录结构

|类型|描述|	默认位置|设置|
|---|---|---|---|
|Home||Elasticsearch主目录或 |$ES_HOME|通过解压缩归档创建的目录
|bin|二进制脚本，包括elasticsearch启动节点和elasticsearch-plugin安装插件|$ES_HOME/bin|
|conf|配置文件包括 elasticsearch.yml|$ES_HOME/config|ES_PATH_CONF
|data|节点上分配的每个索引/分片的数据文件的位置。可以容纳多个位置。|$ES_HOME/data|path.data
|logs|日志文件位置。|$ES_HOME/logs|path.logs
| plugins|插件文件位置。每个插件都将包含在一个子目录中。|$ES_HOME/plugins
|repo|共享文件系统存储库位置。可以容纳多个位置。文件系统存储库可以放在此处指定的任何目录的任何子目录中。|未配置|path.repo

# 2. 配置ElasticSearch

Elasticsearch具有良好的默认值，只需要很少的配置。可以使用[Cluster Update Settings API](https://www.elastic.co/guide/en/elasticsearch/reference/6.5/cluster-update-settings.html) 在正在运行的群集上更改大多数设置 。

配置文件应包含特定于节点的设置（例如node.name和路径），或节点为了能够加入群集而需要的设置，例如`cluster.name`和`network.host。`

## 2.1. 配置文件位置

Elasticsearch有三个配置文件：

- elasticsearch.yml 用于配置Elasticsearch
- jvm.options 用于配置Elasticsearch JVM设置
- log4j2.properties 用于配置Elasticsearch日志记录

这些文件位于config目录中，其默认位置取决于安装是来自存档分发（tar.gz或 zip）还是包分发（Debian或RPM软件包）

对于归档分发，config目录位置默认为 `$ES_HOME/config`。可以通过ES_PATH_CONF环境变量更改config目录的位置， 如下所示：

	ES_PATH_CONF=/path/to/my/config ./bin/elasticsearch

或者，您可以通过命令行或通过shell配置文件来export获取ES_PATH_CONF环境变量。

对于包分发，config目录位置默认为 `/etc/elasticsearch`。`config`目录的位置也可以通过`ES_PATH_CONF`环境变量进行更改，但请注意，在`shell`中设置它是不够的。相反，此变量来自 `/etc/default/elasticsearch`（对于Debian软件包）和 `/etc/sysconfig/elasticsearch`（对于RPM软件包）。您需要相应地编辑`ES_PATH_CONF=/etc/elasticsearch`其中一个文件中的 条目以更改配置目录位置。

## 2.2. 配置文件格式

配置格式为YAML。以下是更改数据路径和日志目录的示例：

	path:
	    data: /var/lib/elasticsearch
	    logs: /var/log/elasticsearch

设置也可以按如下方式展平：

	path.data: /var/lib/elasticsearch
	path.logs: /var/log/elasticsearch

## 2.3. 环境变量

使用`${...}`配置文件中的符号引用的环境变量将替换为环境变量的值，例如：

	node.name:    ${HOSTNAME}
	network.host: ${ES_NETWORK_HOST}


对于您不希望存储在配置文件中的设置，可以使用该值`${prompt.text}`或`${prompt.secret}`在前台启动Elasticsearch。`${prompt.secret}`已禁用回显，以便输入的值不会显示在您的终端中; `${prompt.text}`将允许您在键入时查看值。例如：
	
	node:
	  name: ${prompt.text}

启动Elasticsearch时，系统会提示您输入实际值，如下所示：

	Enter value for [node.name]:

