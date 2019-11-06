---
layout: post
title: "Django 处理Request过程 的 中间件"
date: 2018-03-07 16:34:52 +0800
category: Python
tags: [Python, Django]
---
* content
{:toc}

# Django 处理Request过程 的 中间件



> Django的每一个请求都是先通过中间件中的 process_request 函数，这个函数返回 None 或者 HttpResponse 对象，如果返回前者，继续处理其它中间件，如果返回一个 HttpResponse，就处理中止，返回到网页上。
> 

## 中间件执行过程

	HttpRequest -> process_request -> process_view -> view function 
	-> process_exception? ->  process_template_response? -> process_response
	
1. Process 预处理函数： `process_request(self, request)` 

	这个方法的调用时机在Django接收到request之后，但仍未解析URL以确定当前运行的师徒函数。 Django向他传入相应的Request对象，以便在方法中修改。
	
	如果返回None， Django将继续处理这个request， 执行后续的中间件，然后调用相应的view。
	
	如果返回HttpResponse对象，Django将不再执行任何除了`process_response`以外其它的中间件以及相应的view， Django将立即返回该`HttpResponse`
	
	
2. View预处理函数： `process_view(self, request, callback, callback_args, callback_kwargs)` 

	这个方法的调用时机在 Django 执行完 request 预处理函数并确定待执行的 view （即callback参数）之后，但在 view 函数实际执行之前。
	
	request：HttpRequest 对象。

	callback：Django将调用的处理request的python函数. 这是实际的函数对象本身, 而不是字符串表述的函数名。
	
	args：将传入view的位置参数列表，但不包括request参数(它通常是传入view的第一个参数)。

	kwargs：将传入view的关键字参数字典。

	process_view() 应当返回None或 HttpResponse 对象。如果返回 None， Django将继续处理这个request ，执行后续的中间件， 然后调用相应的view。
	
	如果返回 HttpResponse 对象，Django 将不再执行任何其它的中间件(不论种类)以及相应的view，Django将立即返回。
	
3. Template模版渲染函数：`process_template_response()`

	默认不执行，只有在视图函数的返回结果对象中有render方法才会执行，并把对象的render方法的返回值返回给用户（注意不返回视图函数的return的结果了，而是返回视图函数 return值（对象）中rende方法的结果）

	Exception后处理函数:process_exception(self, request, exception)
这个方法只有在 request 处理过程中出了问题并且view 函数抛出了一个未捕获的异常时才会被调用。这个钩子可以用来发送错误通知，将现场相关信息输出到日志文件，或者甚至尝试从错误中自动恢复。
	
	这个函数的参数除了一贯的request对象之外，还包括view函数抛出的实际的异常对象exception 。

4. `process_exception()` 应当返回None或HttpResponse对象。
	
	如果返回None，Django将用框架内置的异常处理机制继续处理相应request。
如果返回HttpResponse对象，Django将使用该response对象，而短路框架内置的异常处理机制。

5. Response后处理函数:`process_response(self, request, response)`

	这个方法的调用时机在 Django 执行 view 函数并生成 response 之后。
该处理器能修改response 的内容；一个常见的用途是内容压缩，如gzip所请求的HTML页面。
这个方法的参数相当直观：request是request对象，而response则是从view中返回的response对象。

	process_response() 必须返回 HttpResponse 对象. 这个 response 对象可以是传入函数的那一个原始对象（通常已被修改），也可以是全新生成的。


## 自定义中间件：

1. process_response一定要有reurn否则会报错，自定义的中间件response方法没有return，会交给下一个中间件，导致http请求中断了。

2. `process_view（self, request, callback,callback_args,callback_kwargs）`方法介绍：

	- 执行完所有中间件的request方法
	- url匹配成功
	- 拿到视图函数的名称、参数，（注意不执行）再执行process_view()方法
	- 最后去执行视图函数

	既然process_view 拿到视图函数的名称、参数，（不执行） 再执行process_view()方法，最后才去执行视图函数，所以可以在执行process_view环节时直接把函数执行进行return返回。注意：此时的process_view2将不再被调用执行，而是直接跳转到最后一个中间件， 执行最后一个中间件的response方法，逐步返回。

3. process_template_response方法默认也不会执行，只有在视图函数的返回对象中有render方法才会执行.并把对象的render方法的返回值返回给用户（注意不返回视图函数的return的结果了，而是返这个结果对象里render方法的返回值，并且process_template_response方法也要返回response不然会报错。

4. 添加process_exception方法后发现在函数正常执行情况下该方法不会被调用
	- 执行完所有 request 方法；
	- 执行所有 process_view方法；
	- 如果视图函数出错，执行process_exception方法，如果第一个中间件的process_exception方法有了返回值就不再执行其他中间件process_exception，直接执行response方法；
	- 执行所有response方法；
最后返回process_exception的返回值


