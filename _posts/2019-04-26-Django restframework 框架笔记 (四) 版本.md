---
layout: post
title: "Django restframework 框架笔记 (四) 版本"
date: 2019-04-26 18:02:48 +0800
category: Django
tags: [Django]
---
* content
{:toc}


# 1. 版本

## 1.1. 基于url param的版本

1. models.py

   ```python
   from django.db import models
   
   # Create your models here.
   
   class Book(models.Model):
       title = models.CharField(max_length=32)
       price = models.DecimalField(max_digits=5, decimal_places=2)
       objects = models.Manager()
   
       class Meta:
           db_table = "old_boy_book"
           verbose_name = "书籍"
           verbose_name_plural = verbose_name
   
       def __str__(self):
           return "《%s》" % self.title
   ```

2. views.py

   ```python
   from rest_framework import status
   from rest_framework.response import Response
   from rest_framework.views import APIView
   from rest_framework.versioning import QueryParameterVersioning
   from . import models
   
   class Book(APIView):
       versioning_class = QueryParameterVersioning
       
       def get(self, request, *args, **kwargs):
           # 通过 request.version 获取version信息
           ret = {'code': 1000, 'msg': 'ok', 'results': None, 'version': request.version}
   
           pk = kwargs.get('pk')
           if pk is not None:
               book_obj = models.Book.objects.get(pk=pk)
               ret['results'] = {'title': book_obj.title, 'price': book_obj.price}
               return Response(ret,status=status.HTTP_200_OK)
   
           book_dict = [book for book in models.Book.objects.filter().values()]
           ret['results'] = book_dict
           return Response(ret,status=status.HTTP_200_OK)
   
   ```

3. urls.py

   ```python
   from django.urls import re_path
   
   from . import views
   
   urlpatterns = [
       re_path(r'book/$', views.Book.as_view()),
       re_path(r'book/(?P<pk>.*)/$', views.Book.as_view()),
   ]
   ```

4.  settings.py

   ```python
   REST_FRAMEWORK = {
       # 认证处理
       'DEFAULT_AUTHENTICATION_CLASSES': ['api.utils.auth.Authentication'],
       # 权限处理
       'DEFAULT_PERMISSION_CLASSES': ['api.utils.premission.VIPPremission'],
       # 节流处理
       'DEFAULT_THROTTLE_CLASSES': ['api.utils.throttle.UserThrottle'],
       # 具体节流配置
       'DEFAULT_THROTTLE_RATES': {
           'user': '30/m',
       },
       # 版本控制
       'DEFAULT_VERSION': 'v1',
       'ALLOWED_VERSIONS': ['v1', 'v2'],
       'VERSION_PARAM': 'version',
   }
   ```

5. 测试

   - 合法版本信息

     ```shell
      curl "http://127.0.0.1:8000/api/book/1/?version=v1" 
     {"code":1000,"msg":"ok","results":{"title":"葵花宝典","price":1.23},"version":"v1"}%    
     ```

     

   - 非法版本信息

     ```shell
     curl "http://127.0.0.1:8000/api/book/1/?version=v3" 
     {"detail":"Invalid version in query parameter."}% 
     ```

     

   

## 1.2. 在URL Path中获取

1. url.py

   ```python
   from django.urls import re_path
   
   from . import views
   
   urlpatterns = [
       re_path(r'(?P<version>[v1|v2])/book/$', views.Book.as_view()),
   ]
   ```

2. views.py

   ```python
   from rest_framework import status
   from rest_framework.response import Response
   from rest_framework.views import APIView
   from rest_framework.versioning import URLPathVersioning
   from rest_framework.permissions import AllowAny
   from . import models
   
   
   # Create your views here.
   
   class Book(APIView):
       versioning_class = URLPathVersioning
       permission_classes = [AllowAny]
       authentication_classes = []
   
       def get(self, request, *args, **kwargs):
           ret = {'code': 1000, 'msg': 'ok', 'results': None, 'version': request.version}
   
           pk = kwargs.get('pk')
           if pk is not None:
               book_obj = models.Book.objects.get(pk=pk)
               ret['results'] = {'title': book_obj.title, 'price': book_obj.price}
               return Response(ret, status=status.HTTP_200_OK)
   
           book_dict = [book for book in models.Book.objects.filter().values()]
           ret['results'] = book_dict
           return Response(ret, status=status.HTTP_200_OK)
   ```

   

3. settings.py

   ```python
   REST_FRAMEWORK = {
       # 认证处理
       'DEFAULT_AUTHENTICATION_CLASSES': ['api.utils.auth.Authentication'],
       # 权限处理
       'DEFAULT_PERMISSION_CLASSES': ['api.utils.premission.VIPPremission'],
       # 节流处理
       'DEFAULT_THROTTLE_CLASSES': ['api.utils.throttle.UserThrottle'],
       # 具体节流配置
       'DEFAULT_THROTTLE_RATES': {
           'user': '30/m',
       },
       # 版本控制
       # 'DEFAULT_VERSION': 'v1',
       # 'ALLOWED_VERSIONS': ['v1', 'v2'],
       # 'VERSION_PARAM': 'version',
   }
   ```

4. 测试

   - 合法请求

     ```shell
     $ curl "http://127.0.0.1:8000/api/v1/book/"
     
     {"code":1000,"msg":"ok","results":[{"id":1,"title":"葵花宝典","price":1.23},{"id":2,"title":"九阴真经","price":3.2},{"id":3,"title":"如来神掌","price":3.23},{"id":4,"title":"玉女心经","price":3.1}],"version":"1"}%
     ```

     

   - 非法请求

     ```shell
     $ curl -I "http://127.0.0.1:8000/api/v3/book/"
     HTTP/1.1 404 Not Found
     ```

5. 配置到全局当中

   `settings.py`

   ```python
   
   REST_FRAMEWORK = {
       # 认证处理
       'DEFAULT_AUTHENTICATION_CLASSES': ['api.utils.auth.Authentication'],
       # 权限处理
       'DEFAULT_PERMISSION_CLASSES': ['api.utils.premission.VIPPremission'],
       # 节流处理
       'DEFAULT_THROTTLE_CLASSES': ['api.utils.throttle.UserThrottle'],
       # 具体节流配置
       'DEFAULT_THROTTLE_RATES': {
           'user': '30/m',
       },
       # 版本控制
       # 'DEFAULT_VERSION': 'v1',
       # 'ALLOWED_VERSIONS': ['v1', 'v2'],
       # 'VERSION_PARAM': 'version',
       'DEFAULT_VERSIONING_CLASS': 'rest_framework.versioning.URLPathVersioning',
   }
   ```

   