---
layout: post
title: "ElasticSearch 整理(一) 基础"
date: 2018-03-28 14:55:18 +0800
category: ELK
tags: [ELK]
---
* content
{:toc}

# 1. 介绍

Elasticsearch是一个高可扩展的、开源的、全文本搜索和分析的引擎。它允许你近乎实时地存储，检索和分析大量数据。它通常用作底层引擎/技术，为具有复杂搜索特性和需求的应用程序提供动力。

# 2. 基本概念

## 2.1. Near Realtime (NRT) 实时

ELasticSearch 是一个近实时搜索平台。这意味着从索引文档到搜索到文档的时间有一点延迟（通常是1秒）

## 2.2. Cluster 集群

集群是一个或多个节点（服务器）的集合，它们共同保存您的整个数据，并提供跨所有节点的联合索引和搜索功能。群集由唯一名称标识，默认情况下为“`elasticsearch`”。此名称很重要，因为如果节点设置为按名称加入群集，则该节点只能是群集的一部分。

确保不要在不同的环境中重用相同的群集名称，否则最终会导致节点加入错误的群集。例如，您可以使用`logging-dev`，`logging-stage`以及`logging-prod` 用于开发，登台和生产集群。

请注意，拥有一个只包含单个节点的集群是完全正常的。此外，您还可以拥有多个独立的集群，每个集群都有自己唯一的集群名称。


## 2.3. 节点

节点是作为群集一部分的单个服务器，存储数据并参与群集的索引和搜索功能。与集群一样，节点由名称标识，默认情况下，该名称是在启动时分配给节点的随机通用唯一标识符（UUID）。如果不需要默认值，可以定义所需的任何节点名称。此名称对于管理目的非常重要，您可以在其中识别网络中的哪些服务器与Elasticsearch集群中的哪些节点相对应。

可以将节点配置为按群集名称加入特定群集。默认情况下，每个节点都设置为加入一个名为cluster的集群elasticsearch，这意味着如果您在网络上启动了许多节点并且假设它们可以相互发现 - 它们将自动形成并加入一个名为的集群elasticsearch。

在单个群集中，您可以拥有任意数量的节点。此外，如果您的网络上当前没有其他Elasticsearch节点正在运行，则默认情况下启动单个节点将形成一个名为的新单节点集群elasticsearch。

## 2.4. 索引

索引是具有某些类似特征的文档集合。例如，您可以拥有客户数据的索引，产品目录的另一个索引以及订单数据的另一个索引。索引由名称标识（必须全部为小写），并且此名称用于在对其中的文档执行索引，搜索，更新和删除操作时引用索引。

在单个群集中，您可以根据需要定义任意数量的索引。

## 2.5. 类型（6.0.0 以后弃用）

一种类型，曾经是索引的逻辑类别/分区，允许您在同一索引中存储不同类型的文档，例如一种类型用于用户，另一种类型用于博客帖子。不再可能在索引中创建多个类型，并且将在更高版本中删除类型的整个概念。

## 2.6. 文档

文档是可以编制索引的基本信息单元。例如，您可以为单个客户提供文档，为单个产品提供另一个文档，为单个订单提供另一个文档。该文档以JSON（JavaScript Object Notation）表示，JSON是一种普遍存在的互联网数据交换格式。

在索引/类型中，您可以根据需要存储任意数量的文档。请注意，尽管文档实际上位于索引中，但实际上必须将文档编入索引/分配给索引中的类型。

## 2.7. 碎片和副本

索引可能存储大量可能超过单个节点的硬件限制的数据。例如，占独从单个节点提供搜索请求。

为了解决这个问题，Elasticsearch提供了将索引细分为多个称为分片的功能。创建索引时，只需定义所需的分片数即可。每个分片本身都是一个功能齐全且独立的“索引”，可以托管在集群中的任何节点上。

分片很重要，主要有两个原因：

它允许您水平分割/缩放内容量
它允许您跨分片（可能在多个节点上）分布和并行化操作，从而提高性能/吞吐量
分片的分布方式以及如何将文档聚合回搜索请求的机制完全由Elasticsearch管理，对用户来说是透明的。

在任何时候都可以预期出现故障的网络/云环境中，非常有用，强烈建议使用故障转移机制，以防分片/节点以某种方式脱机或因任何原因消失。为此，Elasticsearch允许您将索引的分片的一个或多个副本制作成所谓的副本分片或简称副本。

复制很重要，主要有两个原因：

它在碎片/节点出现故障时提供高可用性。因此，请务必注意，副本分片永远不会在与从中复制的原始/主分片相同的节点上分配。
它允许您扩展搜索量/吞吐量，因为可以在所有副本上并行执行搜索。
总而言之，每个索引可以拆分为多个分片。索引也可以复制为零（表示没有副本）或更多次。复制后，每个索引都将具有主分片（从中复制的原始分片）和副本分片（主分片的副本）。

可以在创建索引时为每个索引定义分片和副本的数量。创建索引后，您还可以随时动态更改副本数。您可以使用_shrink和_splitAPI 更改现有索引的分片数，但这不是一项简单的任务，预先计划正确数量的分片是最佳方法。

默认情况下，Elasticsearch中的每个索引都分配了5个主分片和1个副本，这意味着如果群集中至少有两个节点，则索引将包含5个主分片和另外5个副本分片（1个完整副本），总计为每个索引10个分片



# 3. 安装

## 3.1. 环境&安装

1. 配置 Java 环境
	
	Elasticsearch 至少需要Java 8.建议您使用Oracle JDK版本1.8.0_131.
	
	在安装Elasticsearch之前，请先运行检查Java版本,如果安装了 openjdk 先卸载 openjdk
	
		$ rpm -qa |grep java
		$ yum remove -y java-1.8.0-openjdk-1.8.0.201.b09-2.el7_6.x86_64
	
	下载安装 Oracle JDK，连接地址 [Oracle 下载地址](https://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html)
	
		$ java -version
		java version "1.8.0_102"
		Java(TM) SE Runtime Environment (build 1.8.0_102-b14)
		Java HotSpot(TM) 64-Bit Server VM (build 25.102-b14, mixed mode)
		
		$ echo $JAVA_HOME 
		/usr/local/jdk1.8.0_102

2. 下载 ElasticSearch

		$ wget -O es.tgz https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-6.5.4.tar.gz  
		$ tar zxf es.tgz 
		$ mv elasticsearch-6.5.4 /usr/local/es
	
3. 启动 ElasticSearch

		$ cd /usr/local/es 
		$ ./bin/elasticsearch 
		# 此时会报错，因为 ElasticSearch 默认不允许用 root 用户启动
		# 创建用户
		$ groupadd es
		$ useradd es -g es -d /usr/local/es -p 123456
		$ chown -R es:es /usr/local/es
		# 使用 es 用户启动 ElasticSearch
		$ su - es "/usr/local/es/bin/elasticsearch"  

4. 检查是否正常启动

		$  curl http://localhost:9200
		{
		  "name" : "z1QFwRy",
		  "cluster_name" : "elasticsearch",
		  "cluster_uuid" : "TqCIxTJ8SsSYMRJdumIzLA",
		  "version" : {
		    "number" : "6.5.4",
		    "build_flavor" : "default",
		    "build_type" : "tar",
		    "build_hash" : "d2ef93d",
		    "build_date" : "2018-12-17T21:17:40.758843Z",
		    "build_snapshot" : false,
		    "lucene_version" : "7.5.0",
		    "minimum_wire_compatibility_version" : "5.6.0",
		    "minimum_index_compatibility_version" : "5.0.0"
		  },
		  "tagline" : "You Know, for Search"
		}
		

到此，ElasticSearch 单机版就按照完成。

# 4. 基本使用介绍

## 4.1. 集群健康

让我们从基本运行状况检查开始，我们可以使用它来查看集群的运行情况。我们将使用 `curl` 来执行此操作，但您可以使用任何允许您进行 `HTTP/REST` 调用的工具。假设我们仍然在我们启动 Elasticsearch 的同一节点上打开另一个命令shell窗口。

要检查群集运行状况，我们将使用 `_cat` API。

	$ curl -XGET http://127.0.0.1:9200/_cat/health\?v   
	epoch      timestamp cluster       status node.total node.data shards pri relo init unassign pending_tasks max_task_wait_time active_shards_percent
	1553783069 14:24:29  elasticsearch green           1         1      0   0    0    0        0             0                  -                100.0%

可以看到 集群名称 `elasticsearch`，状态 绿色，节点总数等等信息

集群健康状态分为三种颜色：

- 绿色，一切都是正常情况
- 黄色，所有数据都可用，但尚未分配一些副本（群集功能齐全）
- 红色，某些数据由于某种原因不可用（群集部分功能）

**注意：**

当群集为红色时，它将继续提供来自可用分片的搜索请求，但您可能需要尽快修复它，因为存在未分配的分片。

同样从上面的响应中，我们可以看到总共1个节点，并且我们有0个分片，因为我们还没有数据。请注意，由于我们使用默认群集名称（elasticsearch），并且由于Elasticsearch默认使用单播网络发现来查找同一台计算机上的其他节点，因此您可能会意外启动计算机上的多个节点并拥有它们所有人都加入一个集群。在这种情况下，您可能会在上面的响应中看到多个节点。

我们还可以获得群集中的节点列表，如下所示：

	$ curl -XGET http://127.0.0.1:9200/_cat/nodes\?v
	ip        heap.percent ram.percent cpu load_1m load_5m load_15m node.role master name
	127.0.0.1           32          93   0    0.00    0.01     0.05 mdi       *      z1QFwRy

## 4.2. 列出所有指数

	$ curl -XGET http://127.0.0.1:9200/_cat/indices\?v
	health status index uuid pri rep docs.count docs.deleted store.size pri.store.size

这仅仅意味着我们在集群中还没有索引。

## 4.3. 创建索引

现在让我们创建一个名为“customer”的索引，然后再次列出所有索引：

	$ curl -XPUT http://127.0.0.1:9200/customer\?pretty
	{
	  "acknowledged" : true,
	  "shards_acknowledged" : true,
	  "index" : "customer"
	}

第一个命令使用PUT动词创建名为“customer”的索引。我们只是追加pretty到调用的末尾，告诉它打印JSON响应（如果有的话）。

	$ curl -XGET http://127.0.0.1:9200/_cat/indices\?v 
	health status index    uuid                   pri rep docs.count docs.deleted store.size pri.store.size
	yellow open   customer 7-4DBNarT9GRLQEPIHRcsg   5   1          0            0      1.1kb          1.1kb

第二个命令的结果告诉我们，我们现在有一个名为customer的索引，它有5个主分片和1个副本（默认值），并且它包含0个文档。

您可能还注意到客户索引标记了黄色运行状况。回想一下我们之前的讨论，黄色表示某些副本尚未（尚未）分配。此索引发生这种情况的原因是因为默认情况下Elasticsearch为此索引创建了一个副本。由于我们目前只有一个节点在运行，因此在另一个节点加入集群的稍后时间点之前，尚无法分配一个副本（用于高可用性）。将该副本分配到第二个节点后，此索引的运行状况将变为绿色。

## 4.4. 索引查询和文档

现在让我们在客户索引中加入一些内容。我们将一个简单的客户文档索引到客户索引中，ID为1，如下所示：
	
	$ curl -X PUT "localhost:9200/customer/_doc/1?pretty" -H 'Content-Type: application/json' -d'
	{
	  "name": "John Doe"
	}
	'

返回：

	{
	  "_index" : "customer",
	  "_type" : "_doc",
	  "_id" : "1",
	  "_version" : 1,
	  "result" : "created",
	  "_shards" : {
	    "total" : 2,
	    "successful" : 1,
	    "failed" : 0
	  },
	  "_seq_no" : 0,
	  "_primary_term" : 1
	}

从上面可以看出，在客户索引中成功创建了一个新的客户文档。该文档还具有我们在索引时指定的内部标识1。

值得注意的是，Elasticsearch不需要在将文档编入索引之前先显式创建索引。在前面的示例中，如果客户索引事先尚未存在，则Elasticsearch将自动创建客户索引。

我们现在检索刚刚编入索引的文档：

	$ curl -X GET "localhost:9200/customer/_doc/1?pretty"

返回：

	{
	  "_index" : "customer",
	  "_type" : "_doc",
	  "_id" : "1",
	  "_version" : 1,
	  "found" : true,
	  "_source" : {
	    "name" : "John Doe"
	  }
	}

除了字段之外，这里没有任何异常 found，声明我们找到了一个具有请求的ID 1和另一个字段的_source文档，它返回我们从上一步索引的完整JSON文档。

## 4.5. 删除索引

现在让我们删除刚刚创建的索引，然后再次列出所有索引：

	$ curl -X DELETE "localhost:9200/customer?pretty"

返回：

	{
	  "acknowledged" : true
	}

执行

	$ curl -X GET "localhost:9200/_cat/indices?v"

返回：

	health status index uuid pri rep docs.count docs.deleted store.size pri.store.size

如果我们仔细研究上述命令，我们实际上可以看到我们如何在Elasticsearch中访问数据的模式。该模式可归纳如下：

	<HTTP Verb> /<Index>/<Type>/<ID>

这种REST访问模式在所有API命令中都非常普遍，如果您能够简单地记住它，您将在掌握Elasticsearch方面有一个良好的开端。

# 5. 修改数据

## 5.1. 更新文档

除了能够索引和替换文档，我们还可以更新文档。请注意，Elasticsearch实际上并没有在引擎盖下进行就地更新。每当我们进行更新时，Elasticsearch都会删除旧文档，然后一次性对应用了更新的新文档编制索引。

此示例显示如何通过将名称字段更改为“Jane Doe”来更新以前的文档（ID为1）：

1. 查看索引 `customer` 中文档 `1`
	
		$ curl -X GET "localhost:9200/customer/_doc/1?pretty"                                           
		{
		  "_index" : "customer",
		  "_type" : "_doc",
		  "_id" : "1",
		  "_version" : 1,
		  "found" : true,
		  "_source" : {
		    "name" : "John LEE"
		  }
		}

2. 修改文档字段

		$ curl -X POST "localhost:9200/customer/_doc/1/_update?pretty" -H 'Content-Type: application/json' -d'
		{
		   "doc": { "name": "Jane Doe" }
		}
		'
	返回值
	
		{
		  "_index" : "customer",
		  "_type" : "_doc",
		  "_id" : "1",
		  "_version" : 2,
		  "result" : "updated",
		  "_shards" : {
		    "total" : 2,
		    "successful" : 1,
		    "failed" : 0
		  },
		  "_seq_no" : 1,
		  "_primary_term" : 1
		}

3. 查询字段是否更新

		$ curl -X GET "localhost:9200/customer/_doc/1?pretty"                                                 
		{
		  "_index" : "customer",
		  "_type" : "_doc",
		  "_id" : "1",
		  "_version" : 2,
		  "found" : true,
		  "_source" : {
		    "name" : "Jane Doe"
		  }
		}

4. 此示例显示如何通过将名称字段更改为“Jane Doe”来更新我们以前的文档（ID为1），同时为其添加年龄字段：

		$ curl -X POST "localhost:9200/customer/_doc/1/_update?pretty" -H 'Content-Type: application/json' -d'
		{
		  "doc": { "name": "Jane Doe", "age": 20 }
		}
		'

	返回值

		{
		  "_index" : "customer",
		  "_type" : "_doc",
		  "_id" : "1",
		  "_version" : 3,
		  "result" : "updated",
		  "_shards" : {
		    "total" : 2,
		    "successful" : 1,
		    "failed" : 0
		  },
		  "_seq_no" : 2,
		  "_primary_term" : 1
		}

5. 也可以使用简单脚本执行更新。此示例使用脚本将年龄增加5：
	
		curl -X POST "localhost:9200/customer/_doc/1/_update?pretty" -H 'Content-Type: application/json' -d'
		{
		  "script" : "ctx._source.age += 5"
		}
		'

	返回值

		{
		  "_index" : "customer",
		  "_type" : "_doc",
		  "_id" : "1",
		  "_version" : 4,
		  "result" : "updated",
		  "_shards" : {
		    "total" : 2,
		    "successful" : 1,
		    "failed" : 0
		  },
		  "_seq_no" : 3,
		  "_primary_term" : 1
		}
	
6. 查看当前文档数据

		$ curl -X GET "localhost:9200/customer/_doc/1?pretty"                                                 
		{
		  "_index" : "customer",
		  "_type" : "_doc",
		  "_id" : "1",
		  "_version" : 4,
		  "found" : true,
		  "_source" : {
		    "name" : "Jane Doe",
		    "age" : 25
		  }
		}

在上面的示例中，ctx._source指的是即将更新的当前源文档。

Elasticsearch提供了在给定查询条件（如SQL UPDATE-WHERE语句）的情况下更新多个文档的功能。请参阅[docs-update-by-query API
](https://www.elastic.co/guide/en/elasticsearch/reference/6.5/docs-update-by-query.html)

## 5.2. 删除文档

删除文档非常简单。此示例显示如何删除ID为2的以前的客户：

	$ curl -X DELETE "localhost:9200/customer/_doc/2?pretty"

请参阅[_delete_by_query API](https://www.elastic.co/guide/en/elasticsearch/reference/6.5/docs-delete-by-query.html)以删除与特定查询匹配的所有文档。值得注意的是，删除整个索引而不是使用Delete By Query API删除所有文档会更有效。

## 5.3. 批量处理

除了能够索引，更新和删除单个文档之外，Elasticsearch还提供了使用 [_bulk API](https://www.elastic.co/guide/en/elasticsearch/reference/6.5/docs-bulk.html) 批量执行上述任何操作的功能。此功能非常重要，因为它提供了一种非常有效的机制，可以尽可能快地进行多个操作，并尽可能少地进行网络往返。

作为一个简单的示例，以下调用在一个批量操作中索引两个文档（ID 1 - John Doe和ID 2 - Jane Doe）：

	$ curl -X POST "localhost:9200/customer/_doc/_bulk?pretty" -H 'Content-Type: application/json' -d'
	{"index":{"_id":"1"}}
	{"name": "John Doe" }
	{"index":{"_id":"2"}}
	{"name": "Jane Doe" }
	'

返回

	{
	  "took" : 339,
	  "errors" : false,
	  "items" : [
	    {
	      "index" : {
	        "_index" : "customer",
	        "_type" : "_doc",
	        "_id" : "1",
	        "_version" : 2,
	        "result" : "updated",
	        "_shards" : {
	          "total" : 2,
	          "successful" : 1,
	          "failed" : 0
	        },
	        "_seq_no" : 1,
	        "_primary_term" : 1,
	        "status" : 200
	      }
	    },
	    {
	      "index" : {
	        "_index" : "customer",
	        "_type" : "_doc",
	        "_id" : "2",
	        "_version" : 1,
	        "result" : "created",
	        "_shards" : {
	          "total" : 2,
	          "successful" : 1,
	          "failed" : 0
	        },
	        "_seq_no" : 0,
	        "_primary_term" : 1,
	        "status" : 201
	      }
	    }
	  ]
	}

此示例更新第一个文档（ID为1），然后在一个批量操作中删除第二个文档（ID为2）：

	$ curl -X POST "localhost:9200/customer/_doc/_bulk?pretty" -H 'Content-Type: application/json' -d'
	{"update":{"_id":"1"}}
	{"doc": { "name": "John Doe becomes Jane Doe" } }
	{"delete":{"_id":"2"}}
	'
请注意，对于删除操作，之后没有相应的源文档，因为删除只需要删除文档的ID。

Bulk API不会因其中一个操作失败而失败。如果单个操作因任何原因失败，它将继续处理其后的其余操作。批量API返回时，它将为每个操作提供一个状态（按照发送的顺序），以便您可以检查特定操作是否失败。

# 6. 探索数据

1. 样本数据

	现在我们已经了解了基础知识，让我们尝试更真实的数据集。我准备了一份关于客户银行账户信息的虚构JSON文档样本。每个文档都有以下架构：
	
		{
		    "account_number": 0,
		    "balance": 16623,
		    "firstname": "Bradshaw",
		    "lastname": "Mckenzie",
		    "age": 29,
		    "gender": "F",
		    "address": "244 Columbus Place",
		    "employer": "Euron",
		    "email": "bradshawmckenzie@euron.com",
		    "city": "Hobucken",
		    "state": "CO"
		}

2. 加载示例数据
	
	您可以从此处下载样本数据集（accounts.json）。将它解压缩到我们当前的目录，然后将它加载到我们的集群中，如下所示：
	
		$ wget https://raw.githubusercontent.com/elastic/elasticsearch/master/docs/src/test/resources/accounts.json
		
		$ curl -H "Content-Type: application/json" -XPOST "localhost:9200/bank/_doc/_bulk?pretty&refresh" --data-binary "@accounts.json"
		
		$ curl "localhost:9200/_cat/indices?v"
		health status index    uuid                   pri rep docs.count docs.deleted store.size pri.store.size
		yellow open   customer wuGl5RzeQuGU6Ov7BPZtLw   5   1          1            0      4.6kb          4.6kb
		yellow open   bank     KSgQl0gvSMmglLAmGaIbMw   5   1       1000            0    482.5kb        482.5kb

这意味着我们只是成功地将1000个文档批量索引到银行索引（在_doc类型下）。


## 6.1. Search API

现在让我们从一些简单的搜索开始吧。有运行检索两种基本方式：一种是通过发送搜索参数 [REST request URI ](https://www.elastic.co/guide/en/elasticsearch/reference/6.5/search-uri-request.html) 和其他通过发送他们 [REST request body](https://www.elastic.co/guide/en/elasticsearch/reference/6.5/search-request-body.html)。请求体方法允许您更具表现力，并以更可读的JSON格式定义搜索。我们将尝试一个请求URI方法的示例，但是对于本教程的其余部分，我们将专门使用请求体方法。

可以从_search端点访问用于搜索的REST API 。此示例返回银行索引中的所有文档：

	$ curl -X GET "localhost:9200/bank/_search?q=*&sort=account_number:asc&pretty"

让我们首先剖析搜索语句。我们正在`_search`银行索引中搜索（端点），该`q=*`参数指示Elasticsearch匹配索引中的所有文档。该`sort=account_number:asc`参数指示使用`account_number`每个文档的字段以升序对结果进行排序。该`pretty`参数再次告诉Elasticsearch返回漂亮的`JSON`结果。

返回：
	
	{
	  "took" : 194,
	  "timed_out" : false,
	  "_shards" : {
	    "total" : 5,
	    "successful" : 5,
	    "skipped" : 0,
	    "failed" : 0
	  },
	  "hits" : {
	    "total" : 1000,
	    "max_score" : null,
	    "hits" : [
	      {
	        "_index" : "bank",
	        "_type" : "_doc",
	        "_id" : "0",
	        "_score" : null,
	        "_source" : {
	          "account_number" : 0,
	          "balance" : 16623,
	          "firstname" : "Bradshaw",
	          "lastname" : "Mckenzie",
	          "age" : 29,
	          "gender" : "F",
	          "address" : "244 Columbus Place",
	          "employer" : "Euron",
	          "email" : "bradshawmckenzie@euron.com",
	          "city" : "Hobucken",
	          "state" : "CO"
	        },
	        "sort" : [
	          0
	        ]
	      },...
	    ]
	   }
	}

至于回复，我们看到以下部分：

- `took` - Elasticsearch执行搜索的时间（以毫秒为单位）
- `timed_out` - 告诉我们搜索是否超时
- `_shards` - 告诉我们搜索了多少个分片，以及搜索成功/失败分片的计数
- `hits` - 搜索结果
- `hits.total` - 符合我们搜索条件的文档总数
- `hits.hits` - 实际的搜索结果数组（默认为前10个文档）
- `hits.sort` - 对结果进行排序键（如果按分数排序则丢失）
- `hits._score` 和 `max_score` - 暂时忽略这些字段

以下是使用替代请求正文方法的上述完全相同的搜索：

	$ curl -X GET "localhost:9200/bank/_search" -H 'Content-Type: application/json' -d'
	{
	  "query": { "match_all": {} },
	  "sort": [
	    { "account_number": "asc" }
	  ]
	}
	'
这里的不同之处在于，我们不是传入`q=*URI`，而是向`_search` API 提供JSON样式的查询请求体

重要的是要理解，一旦您获得了搜索结果，Elasticsearch就完全完成了请求，并且不会在结果中维护任何类型的服务器端资源或打开游标。这与SQL等许多其他平台形成鲜明对比，其中您可能最初预先获得查询结果的部分子集，然后如果要获取（或翻页）其余的则必须连续返回服务器使用某种有状态服务器端游标的结果。

## 6.2. 介绍查询语言

Elasticsearch提供了一种JSON样式的特定于域的语言，可用于执行查询。这被称为[查询DSL](https://www.elastic.co/guide/en/elasticsearch/reference/6.5/query-dsl.html)。查询语言非常全面，乍一看可能令人生畏，但实际学习它的最佳方法是从一些基本示例开始。

回到上一个例子，我们执行了这个查询

	$ curl -X GET "localhost:9200/bank/_search" -H 'Content-Type: application/json' -d'
	{
	  "query": { "match_all": {} }
	}
	'

解析上面的内容，该`query`部分告诉我们查询定义是什么，`match_all`部分只是我们想要运行的查询类型。该`match_all`查询仅仅是在指定索引的所有文件进行搜索。

除了`query`参数，我们还可以传递其他参数来影响搜索结果。在上面我们传入的部分的示例中 `sort`，我们传入`size`：

	curl -X GET "localhost:9200/bank/_search" -H 'Content-Type: application/json' -d'
	{
	  "query": { "match_all": {} },
	  "size": 1
	}
	'

请注意，如果`size`未指定，则默认为10。

此示例执行a `match_all`并返回文档10到19：

	curl -X GET "localhost:9200/bank/_search" -H 'Content-Type: application/json' -d'
	{
	  "query": { "match_all": {} },
	  "from": 10,
	  "size": 10
	}
	'

在`from`（从0开始）参数规定了从启动该文件的索引和`size`参数指定了多少文件，返回从参数开始的。在实现搜索结果的分页时，此功能非常有用。请注意，如果`from`未指定，则默认为0。

此示例执行a `match_all`并按帐户余额降序对结果进行排序，并返回前10个（默认大小）文档。

	curl -X GET "localhost:9200/bank/_search" -H 'Content-Type: application/json' -d'
	{
	  "query": { "match_all": {} },
	  "sort": { "balance": { "order": "desc" } }
	}
	'

## 6.3. 执行搜索

现在我们已经看到了一些基本的搜索参数，让我们再深入研究一下`Query DSL`。我们先来看一下返回的文档字段。默认情况下，完整的JSON文档作为所有搜索的一部分返回。这被称为源（`_source`搜索命中中的字段）。如果我们不希望返回整个源文档，我们只能请求返回源中的几个字段。

此示例显示如何从搜索中返回两个字段`account_number`和`balance`（内部`_source`）：

	curl -X GET "localhost:9200/bank/_search" -H 'Content-Type: application/json' -d'
	{
	  "query": { "match_all": {} },
	  "_source": ["account_number", "balance"]
	}
	'
请注意，上面的示例只是简化了`_source`字段。它仍将只返回一个名为`_source`但在其中的字段，仅包含字段`account_number`并`balance`包含在内

现在让我们转到查询部分。以前，我们已经看到`match_all`查询如何用于匹配所有文档。现在让我们介绍一个名为`match`查询的新查询，它可以被认为是一个基本的现场搜索查询（即针对特定字段或字段集进行的搜索）。

此示例返回编号为20的帐户：

	curl -X GET "localhost:9200/bank/_search" -H 'Content-Type: application/json' -d'
	{
	  "query": { "match": { "account_number": 20 } }
	}
	'

此示例返回地址中包含术语“mill”的所有帐户：

	curl -X GET "localhost:9200/bank/_search" -H 'Content-Type: application/json' -d'
	{
	  "query": { "match": { "address": "mill" } }
	}
	'

此示例返回地址中包含术语“mill”或“lane”的所有帐户：

	curl -X GET "localhost:9200/bank/_search" -H 'Content-Type: application/json' -d'
	{
	  "query": { "match": { "address": "mill lane" } }
	}
	'

此示例是match（match_phrase）的变体，它返回地址中包含短语“mill lane”的所有帐户：

	curl -X GET "localhost:9200/bank/_search" -H 'Content-Type: application/json' -d'
	{
	  "query": { "match_phrase": { "address": "mill lane" } }
	}
	'

我们现在介绍一下这个[bool问题](https://www.elastic.co/guide/en/elasticsearch/reference/6.5/query-dsl-bool-query.html)。该bool查询允许我们使用布尔逻辑将较小的查询组成更大的查询。

此示例组成两个match查询并返回地址中包含“mill”和“lane”的所有帐户：

	curl -X GET "localhost:9200/bank/_search" -H 'Content-Type: application/json' -d'
	{
	  "query": {
	    "bool": {
	      "must": [
	        { "match": { "address": "mill" } },
	        { "match": { "address": "lane" } }
	      ]
	    }
	  }
	}
	'

在上面的示例中，该bool must子句指定必须为true才能将文档视为匹配的所有查询。

相反，此示例组成两个match查询并返回地址中包含“mill”或“lane”的所有帐户：

	curl -X GET "localhost:9200/bank/_search" -H 'Content-Type: application/json' -d'
	{
	  "query": {
	    "bool": {
	      "should": [
	        { "match": { "address": "mill" } },
	        { "match": { "address": "lane" } }
	      ]
	    }
	  }
	}
	'

在上面的示例中，该bool should子句指定了一个查询列表，其中任何一个都必须为true才能使文档被视为匹配。

此示例组成两个match查询并返回地址中既不包含“mill”也不包含“lane”的所有帐户：

	curl -X GET "localhost:9200/bank/_search" -H 'Content-Type: application/json' -d'
	{
	  "query": {
	    "bool": {
	      "must_not": [
	        { "match": { "address": "mill" } },
	        { "match": { "address": "lane" } }
	      ]
	    }
	  }
	}
	'

在上面的示例中，该bool must_not子句指定了一个查询列表，对于要被视为匹配的文档，这些查询都不能为true。

我们可以在查询中同时组合must，should和must_not子句bool。此外，我们可以bool在任何这些bool子句中组合查询来模仿任何复杂的多级布尔逻辑。

此示例返回任何40岁但未居住在ID（aho）中的人的所有帐户：

	curl -X GET "localhost:9200/bank/_search" -H 'Content-Type: application/json' -d'
	{
	  "query": {
	    "bool": {
	      "must": [
	        { "match": { "age": "40" } }
	      ],
	      "must_not": [
	        { "match": { "state": "ID" } }
	      ]
	    }
	  }
	}
	'

## 6.4. 执行过滤器

在上一节中，我们跳过了一个称为文档分数的小细节（`_score`搜索结果中的字段）。分数是一个数值，它是文档与我们指定的搜索查询匹配程度的相对度量。分数越高，文档越相关，分数越低，文档的相关性越低。

但是查询并不总是需要产生分数，特别是当它们仅用于“过滤”文档集时。Elasticsearch检测这些情况并自动优化查询执行，以便不计算无用的分数。

我们在上一节中介绍的bool查询还支持一些`filter`子句，这些子句允许我们使用查询来限制将与其他子句匹配的文档，而不会更改计算分数的方式。作为示例，让我们介绍一下[range查询](https://www.elastic.co/guide/en/elasticsearch/reference/6.5/query-dsl-range-query.html)，它允许我们按一系列值过滤文档。这通常用于数字或日期过滤。

此示例使用bool查询返回所有余额介于20000和30000之间的帐户。换句话说，我们希望找到余额大于或等于20000且小于或等于30000的帐户。

	curl -X GET "localhost:9200/bank/_search" -H 'Content-Type: application/json' -d'
	{
	  "query": {
	    "bool": {
	      "must": { "match_all": {} },
	      "filter": {
	        "range": {
	          "balance": {
	            "gte": 20000,
	            "lte": 30000
	          }
	        }
	      }
	    }
	  }
	}
	'

解析上面的内容，`bool`查询包含`match_all`查询（查询部分）和`range`查询（过滤部分）。我们可以将任何其他查询替换为查询和过滤器部分。在上面的情况下，范围查询非常有意义，因为落入范围的文档都“相同”匹配，即，没有文档比另一文档更相关。

除了`match_all`，`match`，`bool`，和`range`查询，有很多可用的其他查询类型的，我们不会进入他们在这里。由于我们已经基本了解它们的工作原理，因此将这些知识应用于学习和试验其他查询类型应该不会太困难。

## 6.5. 执行聚合

聚合提供了从数据中分组和提取统计信息的功能。考虑聚合的最简单方法是将其大致等同于SQL GROUP BY和SQL聚合函数。在Elasticsearch中，您可以执行返回匹配的搜索，同时在一个响应中返回与命中所有内容分开的聚合结果。这是非常强大和高效的，因为您可以运行查询和多个聚合，并一次性获取两个（或任一）操作的结果，避免使用简洁和简化的API进行网络往返。

首先，此示例按状态对所有帐户进行分组，然后返回按计数降序排序的前10个（默认）状态（也是默认值）：

	curl -X GET "localhost:9200/bank/_search" -H 'Content-Type: application/json' -d'
	{
	  "size": 0,
	  "aggs": {
	    "group_by_state": {
	      "terms": {
	        "field": "state.keyword"
	      }
	    }
	  }
	}
	'

在SQL中，上述聚合在概念上类似于

	SELECT state, COUNT(*) FROM bank GROUP BY state ORDER BY COUNT(*) DESC LIMIT 10;

响应（部分显示）

	{
	  "took": 29,
	  "timed_out": false,
	  "_shards": {
	    "total": 5,
	    "successful": 5,
	    "skipped" : 0,
	    "failed": 0
	  },
	  "hits" : {
	    "total" : 1000,
	    "max_score" : 0.0,
	    "hits" : [ ]
	  },
	  "aggregations" : {
	    "group_by_state" : {
	      "doc_count_error_upper_bound": 20,
	      "sum_other_doc_count": 770,
	      "buckets" : [ {
	        "key" : "ID",
	        "doc_count" : 27
	      }, {
	        "key" : "TX",
	        "doc_count" : 27
	      }, {
	        "key" : "AL",
	        "doc_count" : 25
	      }, {
	        "key" : "MD",
	        "doc_count" : 25
	      }, {
	        "key" : "TN",
	        "doc_count" : 23
	      }, {
	        "key" : "MA",
	        "doc_count" : 21
	      }, {
	        "key" : "NC",
	        "doc_count" : 21
	      }, {
	        "key" : "ND",
	        "doc_count" : 21
	      }, {
	        "key" : "ME",
	        "doc_count" : 20
	      }, {
	        "key" : "MO",
	        "doc_count" : 20
	      } ]
	    }
	  }
	}

我们可以看到ID（爱达荷州）有27个账户，其次是TX（德克萨斯州）的27个账户，其次是AL（阿拉巴马州）的25个账户，依此类推。

请注意，我们设置`size=0`为不显示搜索匹配，因为我们只想在响应中看到聚合结果。

在前一个聚合的基础上，此示例按州计算平均帐户余额（同样仅针对按降序排序的前10个州）：

	curl -X GET "localhost:9200/bank/_search" -H 'Content-Type: application/json' -d'
	{
	  "size": 0,
	  "aggs": {
	    "group_by_state": {
	      "terms": {
	        "field": "state.keyword"
	      },
	      "aggs": {
	        "average_balance": {
	          "avg": {
	            "field": "balance"
	          }
	        }
	      }
	    }
	  }
	}
	'

请注意我们如何嵌套`average_balance`聚合内的`group_by_state`聚合。这是所有聚合的常见模式。您可以在聚合中任意嵌套聚合，以从数据中提取所需的轮转摘要。

在前一个聚合的基础上，我们现在按降序排列平均余额：

	curl -X GET "localhost:9200/bank/_search" -H 'Content-Type: application/json' -d'
	{
	  "size": 0,
	  "aggs": {
	    "group_by_state": {
	      "terms": {
	        "field": "state.keyword",
	        "order": {
	          "average_balance": "desc"
	        }
	      },
	      "aggs": {
	        "average_balance": {
	          "avg": {
	            "field": "balance"
	          }
	        }
	      }
	    }
	  }
	}
	'

此示例演示了我们如何按年龄段（20-29岁，30-39岁和40-49岁）进行分组，然后按性别进行分组，最后得到每个年龄段的平均帐户余额：

	curl -X GET "localhost:9200/bank/_search" -H 'Content-Type: application/json' -d'
	{
	  "size": 0,
	  "aggs": {
	    "group_by_age": {
	      "range": {
	        "field": "age",
	        "ranges": [
	          {
	            "from": 20,
	            "to": 30
	          },
	          {
	            "from": 30,
	            "to": 40
	          },
	          {
	            "from": 40,
	            "to": 50
	          }
	        ]
	      },
	      "aggs": {
	        "group_by_gender": {
	          "terms": {
	            "field": "gender.keyword"
	          },
	          "aggs": {
	            "average_balance": {
	              "avg": {
	                "field": "balance"
	              }
	            }
	          }
	        }
	      }
	    }
	  }
	}
	'

还有许多其他聚合功能，我们在此不再详述。如果您想进行进一步的实验，[聚合参考指南](https://www.elastic.co/guide/en/elasticsearch/reference/6.5/search-aggregations.html)是一个很好的起点。

# 7. 对比


|ElasticSearch|RDBS|
|---|---|
|索引(index)|数据库(database)
|类型(type)|表(table)
|文档(document)|行(row)
|字段(field)|列(column)
|映射(mapping)|表结构(schema)
|查询DSL|SQL
|GET|select
|PUT/POST|update
|DELETE|delete



