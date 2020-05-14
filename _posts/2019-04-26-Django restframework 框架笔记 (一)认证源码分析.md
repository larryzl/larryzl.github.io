---
layout: post
title: "Django restframework 框架笔记 (一)认证源码分析"
date: 2019-04-26 10:38:44 +0800
category: Django
tags: [Django]
---
* content
{:toc}
# 1. 基础

## 1.1. 安装

```shell
pip install djangorestframework
```

## 1.2.  基础知识

django-rest-framework 源码重到处都是基于 `CBV` 和面向对象的封装

1. 面向对象封装的两大特性 :

   - 把同一类方法封装到类中
   - 将数据封装到对象中

2. CBV

   基于反射实现根据请求方式不同，执行不同方法:

   ```python
       def as_view(cls, **initkwargs):
           """Main entry point for a request-response process."""
           for key in initkwargs:
               if key in cls.http_method_names:
                   raise TypeError("You tried to pass in the %s method name as a "
                                   "keyword argument to %s(). Don't do that."
                                   % (key, cls.__name__))
               if not hasattr(cls, key):
                   raise TypeError("%s() received an invalid keyword %r. as_view "
                                   "only accepts arguments that are already "
                                   "attributes of the class." % (cls.__name__, key))
   
           def view(request, *args, **kwargs):
               self = cls(**initkwargs)
               if hasattr(self, 'get') and not hasattr(self, 'head'):
                   self.head = self.get
               self.setup(request, *args, **kwargs)
               if not hasattr(self, 'request'):
                   raise AttributeError(
                       "%s instance has no 'request' attribute. Did you override "
                       "setup() and forget to call super()?" % cls.__name__
                   )
               return self.dispatch(request, *args, **kwargs)
           view.view_class = cls
           view.view_initkwargs = initkwargs
   ```

   请求过程： `url--> as_view() --> view() --> dispatch()` 



# 2. 实例

## 2.1. settings

创建一个project和一个app

```shell
 django-admin startproject app .
 python manage.py startapp api
```

在 `settings.py` 中添加

```python
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'rest_framework',
    'api',
]
```



## 2.2. 配置url

```python
# urls.py:
from django.contrib import admin
from django.urls import re_path, include

urlpatterns = [
    re_path('admin/', admin.site.urls),
    re_path(r'^api/', include('api.urls'))
]

# api.urls.py:
from django.urls import re_path

from . import views

urlpatterns = [
    re_path(r'auth/$',views.AuthView.as_view())
]

```

## 2.3. models

```python
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

## 2.4. views

```python
from rest_framework.response import Response
from rest_framework.views import APIView
from rest_framework import status

from . import models

def md5(user):
    import hashlib
    import time
    # 当前时间，相当于生成一个随机的字符串
    ctime = str(time.time())
    m = hashlib.md5(bytes(user, encoding='utf-8'))
    m.update(bytes(ctime, encoding='utf-8'))
    return m.hexdigest()


class AuthView(APIView):
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

```

## 2.5.  admin.py

```python
from django.contrib import admin

# Register your models here.
from . import models

admin.site.register(models.UserInfo)

```

## 2.6. 启动项目

创建超级管理员

```shell
python manage.py createsuperuser 
```

提交model数据

```shell
python manage.py makemigrations 
python manage.py migrate       
```

启动项目

```shell
python manage.py runserver
```

打开后天添加测试用户

```shell
http://127.0.0.1:8000/admin/
```

## 2.7. Postman 测试验证

![image-20200426112525296](/Users/lei/Library/Application Support/typora-user-images/image-20200426112525296.png)



# 3. 添加认证

## 3.1. url 

```python
from django.urls import re_path

from . import views

urlpatterns = [
    re_path(r'auth/$', views.AuthView.as_view()),
    re_path(r'order/',views.OrderView.as_view())
]
```

## 3.2. views

```python
from rest_framework import exceptions
from rest_framework import status
from rest_framework.authentication import BaseAuthentication
from rest_framework.response import Response
from rest_framework.views import APIView

from . import models


# Create your views here.

def md5(user):
    import hashlib
    import time
    # 当前时间，相当于生成一个随机的字符串
    ctime = str(time.time())
    m = hashlib.md5(bytes(user, encoding='utf-8'))
    m.update(bytes(ctime, encoding='utf-8'))
    return m.hexdigest()


class AuthView(APIView):
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


class Authentication(BaseAuthentication):
    """认证"""

    def authenticate(self, request):
        # token = request._request.GET.get('token')
        token = request.data.get('token')
        token_obj = models.UserToken.objects.filter(token=token).first()
        if not token_obj:
            raise exceptions.AuthenticationFailed('用户认证失败')
        # 在rest framework内部会将这两个字段赋值给request，以供后续操作使用
        return (token_obj.user, token_obj)

    def authenticate_header(self, request):
        pass


class OrderView(APIView):
    '''订单相关业务'''

    authentication_classes = [Authentication, ]  # 添加认证

    def get(self, request, *args, **kwargs):
        # request.user
        # request.auth
        ret = {'code': 1000, 'msg': None, 'data': None}
        try:
            ret['data'] = ORDER_DICT
        except Exception as e:
            pass
        return Response(ret)

```



## 3.3. Postman 测试

![image-20200426113722006](/Users/lei/Library/Application Support/typora-user-images/image-20200426113722006.png)



# 4. DRF认证过程

## 4.1 `dispatch()`  执行过程

封装 `request` ,`self.initial(request,*args,**kwargs)` 执行认证

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
            # 通过反射判断是什么类型请求
            if request.method.lower() in self.http_method_names:
                handler = getattr(self, request.method.lower(),
                                  self.http_method_not_allowed)
            else:
                handler = self.http_method_not_allowed
            response = handler(request, *args, **kwargs)
				# 抛出异常，交给异常处理模块
        except Exception as exc:
            response = self.handle_exception(exc)
				
        self.response = self.finalize_response(request, response, *args, **kwargs)
        return self.response
```

### 4.1.1. 封装 `request`:

1. `initialize_request() ` 封装 `request`

   ```python
       def initialize_request(self, request, *args, **kwargs):
           """
           Returns the initial request object.
           """
           parser_context = self.get_parser_context(request)
   
           return Request(
               request,
               parsers=self.get_parsers(),
               authenticators=self.get_authenticators(), # return [auth() for auth in self.authentication_classes]
               negotiator=self.get_content_negotiator(),
               parser_context=parser_context
           )
   ```

   

2. `self.get_authenticators()` 

   ```python
       def get_authenticators(self):
           """
           Instantiates and returns the list of authenticators that this view can use.
           """
           return [auth() for auth in self.authentication_classes]
   ```

4. `self.authentication_classes`

   ```python
   class APIView(View):
   
       # The following policies may be set at either globally, or per-view.
   		...
       authentication_classes = api_settings.DEFAULT_AUTHENTICATION_CLASSES
       ...
   ```

### 4.1.2. 认证：

1. `self.initial(request, *args, **kwargs)`

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
           # 实现认证
           self.perform_authentication(request)
           self.check_permissions(request)
           self.check_throttles(request)
   ```

2. `self.perform_authentication(request)`

   ```python
       def perform_authentication(self, request):
           """
           Perform authentication on the incoming request.
   
           Note that if you override this and simply 'pass', then authentication
           will instead be performed lazily, the first time either
           `request.user` or `request.auth` is accessed.
           """
           request.user
   ```

3. `user`

   ```python
       @property
       def user(self):
           """
           Returns the user associated with the current request, as authenticated
           by the authentication classes provided to the request.
           """
           if not hasattr(self, '_user'):
               with wrap_attributeerrors():
                 	# 获取认证对象，进行认证
                   self._authenticate()
           return self._user
   ```

4. `self._authenticate()`

   ```python
       def _authenticate(self):
           """
           Attempt to authenticate the request using each authentication instance
           in turn.
           """
         	# 循环认证类的所有对象
           for authenticator in self.authenticators:
               try:
                 	# 执行认证类的 authenticate() 方法,有3种情况:
                   # 1. 如果authenticate()抛出异常,self._not_authenticated()执行
                   # 2. 有返回值, 必须是: (request.user, request.auth)
                   # 3. 没有返回值，表示当前认证不处理，等待下一个认证处理
                   user_auth_tuple = authenticator.authenticate(self)
               except exceptions.APIException:
                   self._not_authenticated()
                   raise
   						# 返回值的处理
               if user_auth_tuple is not None:
                   self._authenticator = authenticator
                   # 如果有返回值，将登陆用户 与 登陆认证 分布保存到 request.user\request.auth 中
                   self.user, self.auth = user_auth_tuple
                   return
   				# 如果user_auth_tuple 为空，代表认证通过，但是没有登陆用户 与 登陆认证 信息，代表是游客
           self._not_authenticated()
```
   
返回值就是例子中的 `return (token_obj.user, token_obj)`
   
当没有返回值，就执行 `self._not_authenticated()`，相当于匿名用户，没有通过认证
   
   ```python
       def _not_authenticated(self):
           """
           Set authenticator, user & authtoken representing an unauthenticated request.
   
           Defaults are None, AnonymousUser & None.
           """
           self._authenticator = None
   
           if api_settings.UNAUTHENTICATED_USER:
               self.user = api_settings.UNAUTHENTICATED_USER()  #AnonymousUser匿名用户
           else:
               self.user = None
   
           if api_settings.UNAUTHENTICATED_TOKEN:
               self.auth = api_settings.UNAUTHENTICATED_TOKEN() # None
           else:
               self.auth = None
```
   
   

## 4.2. 配置文件

```python
class APIView(View):

    # The following policies may be set at either globally, or per-view.
    renderer_classes = api_settings.DEFAULT_RENDERER_CLASSES
    parser_classes = api_settings.DEFAULT_PARSER_CLASSES
    authentication_classes = api_settings.DEFAULT_AUTHENTICATION_CLASSES
    throttle_classes = api_settings.DEFAULT_THROTTLE_CLASSES
    permission_classes = api_settings.DEFAULT_PERMISSION_CLASSES
    content_negotiation_class = api_settings.DEFAULT_CONTENT_NEGOTIATION_CLASS
    metadata_class = api_settings.DEFAULT_METADATA_CLASS
    versioning_class = api_settings.DEFAULT_VERSIONING_CLASS
```



配置优先级依次为: 

当前类中 > `settings.py` > 默认配置 `APISettings( drf 中的 settings.py)`



**APISettings类:**

```python
class APISettings:
    """
    A settings object, that allows API settings to be accessed as properties.
    For example:

        from rest_framework.settings import api_settings
        print(api_settings.DEFAULT_RENDERER_CLASSES)

    Any setting with string import paths will be automatically resolved
    and return the class, rather than the string literal.
    """
    def __init__(self, user_settings=None, defaults=None, import_strings=None):
        if user_settings:
            self._user_settings = self.__check_user_settings(user_settings)
        self.defaults = defaults or DEFAULTS
        self.import_strings = import_strings or IMPORT_STRINGS
        self._cached_attrs = set()
```

默认回去全局配置文件 `settings.py` 中查找

```python
api_settings = APISettings(None, DEFAULTS, IMPORT_STRINGS)


def reload_api_settings(*args, **kwargs):
    setting = kwargs['setting']
    if setting == 'REST_FRAMEWORK':
        api_settings.reload()
```

setting中`REST_FRAMEWORK`中找

全局配置方法：

`api`文件夹下面新建文件夹`utils`,再新建`auth.py`文件，里面写上认证的类

`settings.py`

```python
REST_FRAMEWORK = {
    # 认证处理
      'DEFAULT_AUTHENTICATION_CLASSES': 'api.utils.auth.Authentication',
}
```



`auth.py`

```python
from rest_framework import exceptions
from api import models
from rest_framework.authentication import BaseAuthentication

class Authentication(BaseAuthentication):
    def authenticate(self, request):
        token = request.data.get('token')
        try:
            token_obj = models.UserToken.objects.get(token=token)
            return (token_obj.user, token_obj)
        except:
            raise exceptions.AuthenticationFailed('用户认证失败')

    def authenticate_header(self, request):
        pass
```



在`settings`里面设置的全局认证，所有业务都需要经过认证，如果想让某个不需要认证，只需要在其中添加下面的代码

```python
authentication_classes = [] 
```

# 5. drf的内置认证

drf 的内置认证模块写在 `authentication.py` 中，所有认证类都要继承 `BaseAuthentication` 类，其中规定必须实现的两个方法

```python
class BaseAuthentication:
    """
    All authentication classes should extend BaseAuthentication.
    """

    def authenticate(self, request):
        """
        Authenticate the request and return a two-tuple of (user, token).
        """
        # 内置的认证类，authenticate方法，如果不自己写，默认则抛出异常
        raise NotImplementedError(".authenticate() must be overridden.")

    def authenticate_header(self, request):
        """
        Return a string to be used as the value of the `WWW-Authenticate`
        header in a `401 Unauthenticated` response, or `None` if the
        authentication scheme should return `403 Permission Denied` responses.
        """
        # authenticate_header方法，作用是当认证失败的时候，返回的响应头
        pass
```



**drf 其他内置认证类:**

```
class BasicAuthentication(BaseAuthentication):
    """
    HTTP Basic authentication against username/password.
    """
  	....

class SessionAuthentication(BaseAuthentication):
    """
    Use Django's session framework for authentication.
    """
		....



class TokenAuthentication(BaseAuthentication):
    """
    Simple token based authentication.

    Clients should authenticate by passing the token key in the "Authorization"
    HTTP header, prepended with the string "Token ".  For example:

        Authorization: Token 401f7ac837da42b97f613d789819ff93537bee6a
    """
		....
		

class RemoteUserAuthentication(BaseAuthentication):
    """
    REMOTE_USER authentication.

    To use this, set up your web server to perform authentication, which will
    set the REMOTE_USER environment variable. You will need to have
    'django.contrib.auth.backends.RemoteUserBackend in your
    AUTHENTICATION_BACKENDS setting
    """
		....
```

