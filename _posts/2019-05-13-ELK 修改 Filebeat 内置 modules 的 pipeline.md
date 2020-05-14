---
layout: post
title: "ELK 修改 Filebeat 内置 modules 的 pipeline"
date: 2019-05-13 18:00:24 +0800
category: ELK
tags: [ELK]
---
* content
{:toc}
# 1. Filebeat 模块集

 filebeat 是一个轻量级的日志采集传输工具，正好弥补了Logstash 的缺点，filebeat 提供了很多 “开箱即用” 的模块。

Elasticsearch 在 5.x以后，支持了 Ingest 模块， 这 也就意味着可以将数据直接用 Filebeat 推送到 Elasticsearch，并让 Elasticsearch 既做解析的事情，又做存储的事情。

Filebeat 既支持自定义文件传输，也支持直接使用模块来传输

本文环境

| 软件          | 版本         | IP           | os       |
| ------------- | ------------ | ------------ | -------- |
| filebeat      | 6.6.2-x86_64 | 192.168.8.13 | centos 7 |
| kibana        | 6.6.2        | 192.168.8.12 | centos 7 |
| elasticsearch | 6.6.2        | 192.168.8.12 | centos 7 |

查看filebeat当前支持的模块:

```shell
$ filebeat modules list
Enabled:
nginx

Disabled:
apache2
auditd
elasticsearch
haproxy
icinga
iis
kafka
kibana
logstash
mongodb
mysql
osquery
postgresql
redis
suricata
system
traefik
```

如果是 rpm 安装的filebeat 默认 模块配置文件存储在 `/usr/share/filebeat/module/`

```shell
$ ls /usr/share/filebeat/module/
apache2  auditd  elasticsearch  haproxy  icinga  iis  kafka  kibana  logstash  mongodb  mysql  nginx  osquery  postgresql  redis  suricata  system  traefik

```

# 2. 使用nginx模块

默认的nginx 模块只支持 nginx 默认日志格式，如下:

```
http {
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
}
```

filebeat 开启 nginx 模块

```shell
$ filebeat modules enable nginx  
$ filebeat modules list
Enabled:
nginx
```

修改nginx 模块配置

```shell
$ cat /etc/filebeat/modules.d/nginx.yml
- module: nginx
  # Access logs
  access:
    enabled: true
    # Set custom paths for the log files. If left empty,
    # Filebeat will choose the paths depending on your OS.
    var.paths: ["/var/logs/nginx/access.log"]

  # Error logs
  error:
    enabled: true
    # Set custom paths for the log files. If left empty,
    # Filebeat will choose the paths depending on your OS.
    #var.paths:
```

启动filebeat

```shell
$ systemctl start filebeat.service
```



# 3. 修改nginx日志格式

现实环境中，我们可能需要更多的日志信息来满足业务需求，默认的nginx日志无法满足我们需求，就需要修改nginx日志格式，如下:

```
log_format  main  '$http_x_forwarded_for - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$remote_addr" $request_time '
                      '$upstream_response_time $request_length "$request_uri"';
```

相比默认格式调整、增加了一些字段

```
192.168.8.2 - - [13/May/2019:18:23:20 +0800] "GET /wap HTTP/1.1" 200 710 "-" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_1) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/13.0.3 Safari/605.1.15" "192.168.8.10" 0.000 - 466 "/wap/?appid=14&host=2"
```

#  4. 修改filebeat 解析规则

```shell
$ vi /usr/share/filebeat/module/nginx/access/manifest.yml

module_version: "1.0"

var:
  - name: paths
    default:
      - /var/log/nginx/access.log*
    os.darwin:
      - /usr/local/var/log/nginx/access.log*
    os.windows:
      - c:/programdata/nginx/logs/*access.log*
# 默认 ingest_pipeline 位置
# ingest_pipeline: ingest/default.json
ingest_pipeline: ingest/nginx-access.json
input: config/nginx-access.yml

machine_learning:
- name: response_code
  job: machine_learning/response_code.json
  datafeed: machine_learning/datafeed_response_code.json
  min_version: 5.5.0
- name: low_request_rate
  job: machine_learning/low_request_rate.json
  datafeed: machine_learning/datafeed_low_request_rate.json
  min_version: 5.5.0
- name: remote_ip_url_count
  job: machine_learning/remote_ip_url_count.json
  datafeed: machine_learning/datafeed_remote_ip_url_count.json
  min_version: 5.5.0
- name: remote_ip_request_rate
  job: machine_learning/remote_ip_request_rate.json
  datafeed: machine_learning/datafeed_remote_ip_request_rate.json
  min_version: 5.5.0
- name: visitor_rate
  job: machine_learning/visitor_rate.json
  datafeed: machine_learning/datafeed_visitor_rate.json
  min_version: 5.5.0

requires.processors:
- name: user_agent
  plugin: ingest-user-agent
- name: geoip
  plugin: ingest-geoip

```

备份默认json文件，修改grok配置

```shell
$ cd /usr/share/filebeat/module/nginx/access
$ cp ingest/default.json ingest/nginx-access.json
$ vi ingest/nginx-access.json

····
 "grok":{
        "field":"message",
        "patterns":[
            ""?%{IP_LIST:nginx.access.remote_ip_list} - %{DATA:nginx.access.user_name} \[%{HTTPDATE:nginx.access.time}\] "%{GREEDYDATA:nginx.access.info}" %{NUMBER:nginx.access.response_code} %{NUMBER:nginx.access.body_sent.bytes} "%{DATA:nginx.access.referrer}" "%{DATA:nginx.access.agent}" %{NUMBER:nginx.access.request_time:float} (%{NUMBER:nginx.upstream_response_time}|-) %{NUMBER:nginx.access.request_length} "%{DATA:nginx.access.request_uri}""
        ],
        "pattern_definitions":{
            "IP_LIST":"%{IP}("?,?\s*%{IP})*"
        },
        "ignore_missing":true
    }
···
```

清空 elastisearch 中的pipeline 配置

```shell
curl -XDELETE http://192.168.8.12:9200/_ingest/pipeline/filebeat-6.6.2-nginx-access-default/
```



记得删除 es 与kibana 中已经产生的数据索引信息

启动filebeat

````shell
systemctl start filebeat	
````



更多内容 查看 [官网 filebeat 模块介绍](https://www.elastic.co/guide/en/beats/filebeat/current/filebeat-modules.html)