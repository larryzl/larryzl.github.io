---
layout: post
title: "Django restframework 框架笔记 (五) 解析器"
date: 2019-04-27 13:55:06 +0800
category: Django
tags: [Django]
---
* content
{:toc}


# 1. 解析器

## 1.1. DRF 默认解析器

> DRF 默认启用了3个解析器,可以通过  `rest_framework/settings.py` 中查看默认解析器
>
> ```python
> 'DEFAULT_PARSER_CLASSES': [
>     'rest_framework.parsers.JSONParser',  # json 格式数据
>     'rest_framework.parsers.FormParser',	# form-urlencodeed 格式
>     'rest_framework.parsers.MultiPartParser' # form data格式
> ],
> ```

​	

- JSON格式数据:

  ![image-20200427140404888](/Users/lei/Library/Application Support/typora-user-images/image-20200427140404888.png)

- form-urlencodeed:

  ![image-20200427140602229](/Users/lei/Library/Application Support/typora-user-images/image-20200427140602229.png)

- form-data:

  ![image-20200427140623471](/Users/lei/Library/Application Support/typora-user-images/image-20200427140623471.png)



1. views.py

   ```python
   from rest_framework import status
   from rest_framework.parsers import FormParser
   from rest_framework.permissions import AllowAny
   from rest_framework.response import Response
   from rest_framework.views import APIView
   
   from api.utils.premission import UserPremission
   from . import models
   
   
   class Book(APIView):
       versioning_class = URLPathVersioning
       permission_classes = [AllowAny]
       authentication_classes = []
       parser_classes = [FormParser] # 定义使用哪种解析器
   
       def get(self, request, *args, **kwargs):
           # return Response('ok')
           ret = {'code': 1000, 'msg': 'ok', 'results': None, 'version': request.version, 'url_path': None}
           print('version:', request.version)
   
           pk = kwargs.get('pk')
           if pk is not None:
   						url_path = request.versioning_scheme.reverse(viewname='book_info', request=request)
   						ret['url_path'] = url_path
               book_obj = models.Book.objects.get(pk=pk)
               ret['results'] = {'title': book_obj.title, 'price': book_obj.price}
               return Response(ret, status=status.HTTP_200_OK)
   
           url_path = request.versioning_scheme.reverse(viewname='book_list', request=request)
           ret['url_path'] = url_path
   
           book_dict = [book for book in models.Book.objects.filter().values()]
           ret['results'] = book_dict
   
           return Response(ret, status=status.HTTP_200_OK)
   
       #
       def post(self, request, *args, **kwargs):
           try:
               title = request.data['title']
               price = request.data['price']
               book_obj = models.Book.objects.create(title=title, price=price)
               book_obj.save()
   
               return Response({'code': 1, 'msg': 'ok', 'results': {'title': book_obj.title, 'price': book_obj.price}})
           except Exception as e:
   
               return Response({'code': 5, 'msg': 'params error'})
   ```



## 1.2. 配置全局解析器

`settings.py`:

```python
REST_FRAMEWORK = {
    # 解析器
    'DEFAULT_PARSER_CLASSES': [
        'rest_framework.parsers.JSONParser',
        'rest_framework.parsers.FormParser',
        'rest_framework.parsers.MultiPartParser',
    ],
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
    'DEFAULT_VERSIONING_CLASS': 'rest_framework.versioning.URLPathVersioning',
    'DEFAULT_VERSION': 'v1',
    'ALLOWED_VERSIONS': ['v1', 'v2'],
    'VERSION_PARAM': 'version',

}
```

