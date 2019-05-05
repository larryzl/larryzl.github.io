---
layout: post
title: "K8S 三种外部访问方式: NodePort LoadBalance Ingress 总结"
date: 2018-04-28 16:17:39 +0800
category: K8S
tags: [K8S]
---
* content
{:toc}


# 1. Cluster IP

ClusterIP 服务是 Kubernetes的默认服务，它在集群内部生成一个服务，供集群内的其他应用访问。外部无法访问。

ClusterIP 服务的 YAML 文件如下：

	apiVersion: v1
	kind: Service
	metadata:  
	  name: my-internal-service
	selector:    
	  app: my-app
	spec:
	  type: ClusterIP
	  ports:  
	  - name: http
	    port: 80
	    targetPort: 80
	    protocol: TCP
	   
我们可以使用 Kubernetes proxy 来访问 ClusterIP




# NodePort

# Loadbalance

# Ingress