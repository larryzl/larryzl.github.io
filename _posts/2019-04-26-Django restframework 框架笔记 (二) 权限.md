---
layout: post
title: "Django restframework 框架笔记 (二) 权限"
date: 2019-04-26 10:38:44 +0800
category: Django
tags: [Django]
---
* content
{:toc}
# 1. 添加权限配置

## 1.1. 创建自定义权限管理文件

在`api/utils`中创建 `premission.py`

```python
from rest_framework.permissions import BasePermission

class VIPPremission(BasePermission):
    message = {'code': 401, 'msg': "只允许SVIP用户访问"}
    def has_permission(self, request, view):
        if request.user.user_type != 3:
            return False
        return True

class UserPremission(BasePermission):
    def has_permission(self, request, view):
        if request.user.user_type == 3:
            return True
        return False
```

## 1.2.  在 `settings.py`中配置全局权限

```python
REST_FRAMEWORK = {
    # 认证处理
    'DEFAULT_AUTHENTICATION_CLASSES': ['api.utils.auth.Authentication'],
    # 权限处理
    'DEFAULT_PERMISSION_CLASSES': ['api.utils.premission.VIPPremission'],
}

```

默认权限为`VIPPremission` 如果不满足该权限，则应该返回权限错误信息

## 1.3. 修改 `views.py` 添加局部权限

 `models.py`:

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


class UserInfo(models.Model):
    USER_TYPE = (
        (1, '普通用户'),
        (2, 'VIP'),
        (3, 'SVIP')
    )

    user_type = models.IntegerField(choices=USER_TYPE)
    username = models.CharField(max_length=32)
    password = models.CharField(max_length=64)

    class Meta:
        # db_table = "user_info"
        verbose_name = "用户"
        verbose_name_plural = verbose_name

    def __str__(self):
        return self.username


class UserToken(models.Model):
    user = models.OneToOneField(UserInfo, on_delete=models.CASCADE)
    token = models.CharField(max_length=64)
```

`views.py`:

```python
from rest_framework import status
from rest_framework.response import Response
from rest_framework.views import APIView

from api.utils.premission import UserPremission
from . import models


# Create your views here.

class Book(APIView):
    # 允许具有UserPremission 用户访问
    permission_classes = [UserPremission]
    
    def get(self, request, *args, **kwargs):
        pk = kwargs.get('pk')
        if pk is not None:
            book_obj = models.Book.objects.get(pk=pk)
            return Response({'code': 1000, 'msg': 'ok', 'results': {'title': book_obj.title, 'price': book_obj.price}})
        book_dict = [book for book in models.Book.objects.filter().values()]
        return Response({'code': 1000, 'msg': 'ok', 'results': book_dict})

    def post(self, request, *args, **kwargs):
        try:
            title = request.data['title']
            price = request.data['price']
            book_obj = models.Book.objects.create(title=title, price=price)
            book_obj.save()

            return Response({'code': 1, 'msg': 'ok', 'results': {'title': book_obj.title, 'price': book_obj.price}})
        except Exception as e:

            return Response({'code': 5, 'msg': 'params error'})

    # def delete(self,request,*args,**kwargs):


def md5(user):
    import hashlib
    import time
    # 当前时间，相当于生成一个随机的字符串
    ctime = str(time.time())
    m = hashlib.md5(bytes(user, encoding='utf-8'))
    m.update(bytes(ctime, encoding='utf-8'))
    return m.hexdigest()


class AuthView(APIView):
    authentication_classes = []
    permission_classes = []

    def post(self, request, *args, **kwargs):
        ret = {'code': 1000, 'msg': None}
        try:
            # user = request._request.POST.get('username')
            user = request.data.get('username')
            # pwd = request._request.POST.get('password')
            pwd = request.data.get('password')
            obj = models.UserInfo.objects.filter(username=user, password=pwd).first()
            if not obj:
                ret['code'] = 1001
                ret['msg'] = '用户名或密码错误'
                return Response(ret, status=status.HTTP_401_UNAUTHORIZED)
            # 为用户创建token
            token = md5(user)
            # 存在就更新，不存在就创建
            models.UserToken.objects.update_or_create(user=obj, defaults={'token': token})
            ret['token'] = token
            return Response(ret, status=status.HTTP_200_OK)
        except Exception as e:
            ret['code'] = 1002
            ret['msg'] = '请求异常'
        return Response(ret)


ORDER_DICT = {
    1: {
        'name': 'apple',
        'price': 15
    },
    2: {
        'name': 'dog',
        'price': 100
    }
}


class OrderView(APIView):
    """订单相关业务"""
		# 根据全局配置，只允许具有svip权限的用户访问67945
    def get(self, request, *args, **kwargs):
        # request.user
        # request.auth
        ret = {'code': 1000, 'msg': None, 'data': None}
        try:
            ret['data'] = ORDER_DICT
        except Exception as e:
            print(e)
        return Response(ret)

```

## 1.4. 添加url

```python
urlpatterns = [
    re_path(r'book/$', views.Book.as_view()),
    re_path(r'book/(?P<pk>.*)/$', views.Book.as_view()),
    re_path(r'auth/$', views.AuthView.as_view()),
    re_path(r'order/', views.OrderView.as_view())
]
```



## 1.5. postman 测试

![image-20200426163859615](/Users/lei/Library/Application Support/typora-user-images/image-20200426163859615.png)

![image-20200426163956078](/Users/lei/Library/Application Support/typora-user-images/image-20200426163956078.png)

# 2. 源码分析

## 2.1. 权限流程

1. `dispatch(self, request, *args, **kwargs):`

   ```python
   def dispatch(self, request, *args, **kwargs):
       """
       `.dispatch()` is pretty much the same as Django's regular dispatch,
       but with extra hooks for startup, finalize, and exception handling.
       """
       self.args = args
       self.kwargs = kwargs
       # 1. 封装request
       # 获取原声request： request._request
       # 获取认证类对象 request.authenticators
       # Request(
       #     request,
       #     parsers=self.get_parsers(),
       #     authenticators=self.get_authenticators(),
       #     negotiator=self.get_content_negotiator(),
       #     parser_context=parser_context
       # )
       request = self.initialize_request(request, *args, **kwargs)
       self.request = request
       self.headers = self.default_response_headers  # deprecate?
   
       try:
           # 2. 认证
           self.initial(request, *args, **kwargs)
   
           # Get the appropriate handler method
           if request.method.lower() in self.http_method_names:
               handler = getattr(self, request.method.lower(),
                                 self.http_method_not_allowed)
           else:
               handler = self.http_method_not_allowed
   
           response = handler(request, *args, **kwargs)
   
       except Exception as exc:
           response = self.handle_exception(exc)
   
       self.response = self.finalize_response(request, response, *args, **kwargs)
       return self.response
   ```

2. `self.initial(request,*args,**kwargs)`

   ```python
   def initial(self, request, *args, **kwargs):
       """
       Runs anything that needs to occur prior to calling the method handler.
       """
       self.format_kwarg = self.get_format_suffix(**kwargs)
   
       # Perform content negotiation and store the accepted info on the request
       neg = self.perform_content_negotiation(request)
       request.accepted_renderer, request.accepted_media_type = neg
   
       # Determine the API version, if versioning is in use.
       version, scheme = self.determine_version(request, *args, **kwargs)
       request.version, request.versioning_scheme = version, scheme
   
       # Ensure that the incoming request is permitted
       self.perform_authentication(request)
       # 检查permission权限
       self.check_permissions(request)
       self.check_throttles(request)
   ```

3. `self.check_permissions(request)`

   ```python
   def check_permissions(self, request):
       """
       Check if the request should be permitted.
       Raises an appropriate exception if the request is not permitted.
       """
       for permission in self.get_permissions():
           if not permission.has_permission(request, self):
               self.permission_denied(
                   request, message=getattr(permission, 'message', None)
               )
   ```

4. `self.get_permissions()`

   ```python
   def get_permissions(self):
       """
       Instantiates and returns the list of permissions that this view requires.
       """
       return [permission() for permission in self.permission_classes]
   ```

5. `self.permission_classes`

   ```python
   class APIView(View):
   
       # The following policies may be set at either globally, or per-view.
       renderer_classes = api_settings.DEFAULT_RENDERER_CLASSES
       parser_classes = api_settings.DEFAULT_PARSER_CLASSES
       authentication_classes = api_settings.DEFAULT_AUTHENTICATION_CLASSES
       throttle_classes = api_settings.DEFAULT_THROTTLE_CLASSES
       # 默认的permission_classes -> api_settions.DEFAULT_PERMISSION_CLASSES
       permission_classes = api_settings.DEFAULT_PERMISSION_CLASSES
       content_negotiation_class = api_settings.DEFAULT_CONTENT_NEGOTIATION_CLASS
       metadata_class = api_settings.DEFAULT_METADATA_CLASS
       versioning_class = api_settings.DEFAULT_VERSIONING_CLASS
   ```

6. `rest_framework/settings.py` 内的`APISettings`

   ```python
   def reload_api_settings(*args, **kwargs):
       setting = kwargs['setting']
       if setting == 'REST_FRAMEWORK':
           api_settings.reload()
   ```

   跟认证一样，在`settings.py`中定义`REST_FRAMEWORK` 为全局配置

   

   ```python
   REST_FRAMEWORK = {
       # 权限处理
       'DEFAULT_PERMISSION_CLASSES': ['api.utils.premission.VIPPremission'],
   }
   ```

## 2.2. 内置权限

`django-rest-framework`内置权限`BasePermission`



```python
class BasePermission(metaclass=BasePermissionMetaclass):
    """
    A base class from which all permission classes should inherit.
    """

    def has_permission(self, request, view):
        """
        Return `True` if permission is granted, `False` otherwise.
        """
        return True

    def has_object_permission(self, request, view, obj):
        """
        Return `True` if permission is granted, `False` otherwise.
        """
        return True

```

自定义权限需要继承 `BasePermission`类，重写 `has_permission`方法

