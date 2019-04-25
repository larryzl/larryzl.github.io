---
layout: post
title: "Python 模块之 queue(转）"
date: 2018-04-01 01:30:57 +0800
category: Python
tags: [Python]
---
* content
{:toc}

原文：[https://blog.csdn.net/GeekLeee/article/details/77883252 
](	原文：https://blog.csdn.net/GeekLeee/article/details/77883252 
)


# 1. 简介

Python的Queue模块提供一种适用于多线程编程的FIFO实现。它可用于在生产者(producer)和消费者(consumer)之间线程安全(thread-safe)地传递消息或其它数据，因此多个线程可以共用同一个Queue实例。Queue的大小（元素的个数）可用来限制内存的使用。


# 2. 使用

## 2.1. Basic FIFO Queue

Queue类实现了一个基本的先进先出(FIFO)容器，使用put()将元素添加到序列尾端，get()从队列尾部移除元素。

	from queue import Queue
	
	q = Queue()
	
	for i in range(3):
	    q.put(i)
	
	while not q.empty():
	    print(q.get())

上例使用单线程演示了元素以插入顺序从队列中移除。结果如下：

	0
	1
	2
	3

## 2.2. LIFO Queue

与标准FIFO实现Queue不同的是，LifoQueue使用后进先出序（会关联一个栈数据结构）。

	from queue import LifoQueue
	
	q = LifoQueue()
	
	for i in range(3):
	    q.put(i)
	
	while not q.empty():
	    print(q.get())

最后put()到队列的元素最先被get()。
	
	2
	1
	0

## 2.3. Priority Queue（优先队列）

除了按元素入列顺序外，有时需要根据队列中元素的特性来决定元素的处理顺序。例如，财务部门的打印任务可能比码农的代码打印任务优先级更高。PriorityQueue依据队列中内容的排序顺序(sort order)来决定那个元素将被检索。

	from queue import PriorityQueue
	
	
	class Job(object):
	    def __init__(self, priority, description):
	        self.priority = priority
	        self.description = description
	        print('New job:', description)
	        return
	
	    def __lt__(self, other):
	        return self.priority < other.priority
	
	q = PriorityQueue()
	
	q.put(Job(5, 'Mid-level job'))
	q.put(Job(10, 'Low-level job'))
	q.put(Job(1, 'Important job'))
	
	while not q.empty():
	    next_job = q.get()
	    print('Processing job', next_job.description)

在这个单线程示例中，job会严格按照优先级从队列中取出。如有有多个线程同时消耗这些job，在get()被调用时，job会依据其优先级被处理。

	New job: Mid-level job
	New job: Low-level job
	New job: Important job
	Processing job: Important job
	Processing job: Mid-level job
	Processing job: Low-level job

## 2.4. Using Queues with Threads

下例通过创建一个简单的播客客户端来展示如何将Queue类和多线程结合使用。这个客户端会从一个或多个RSS源读取内容，先创建一个用于存放下载内容的队列，然后使用多线程并行地处理多个下载任务。

	import time
	from queue import Queue
	from threading import Thread
	
	#: 自己写的解析模块
		
	import feedparser
	
	num_fetch_threads = 2
	enclosure_queue = Queue()
	
	feed_urls = ['http:xxx/xxx',]
	
	
	def downloadEnclosures(i, q):
	    """ 线程worker函数
	    用于处理队列中的元素项，这些守护线程在一个无限循环中，只有当主线程结束时才会结束循环
	    """
	    while True:
	        print('%s: Looking for the next enclosure' % i)
	        url = q.get()
	        print('%s: Downloading: %s' % (i, url))
	        #: 用sleep代替真实的下载
	        time.sleep(i + 2)
	        q.task_done()
	
	for i in range(num_fetch_threads):
	    worker = Thread(target=downloadEnclosures, args=(i, enclosure_queue))
	    worker.setDaemon(True)
	    worker.start()
	
	for url in feed_urls:
	    response = feedparser.parse(url, agent='fetch_podcasts.py')
	    for entry in response['entries']:
	        for enclosure in entry.get('enclosures', []):
	            print('Queuing:', enclosure['url'])
	            enclosure_queue.put(enclosure['url'])
	
	# Now wait for the queue to be empty, indicating that we have
	# processed all of the downloads.
	print('*** Main thread waiting')
	enclosure_queue.join()
	print('*** Done')


# 3. queue 常用方法

方法|解释
|----|----|
queue.qsize()| 返回队列的大小
queue.empty() |如果队列为空，返回True,反之False
queue.full()| 如果队列满了，返回True,反之False
queue.full 与 maxsize| 大小对应
queue.get([block[, timeout]])|获取队列，立即取出一个元素， timeout超时时间
queue.put(item[, timeout]]) |写入队列，立即放入一个元素， timeout超时时间
queue.get_nowait() |相当于queue.get(False)
queue.put_nowait(item)| 相当于queue.put(item, False)
queue.join() |阻塞调用线程，直到队列中的所有任务被处理掉, 实际上意味着等到队列为空，再执行别的操作
queue.task_done() |在完成一项工作之后，queue.task_done()函数向任务已经完成的队列发送一个信号

