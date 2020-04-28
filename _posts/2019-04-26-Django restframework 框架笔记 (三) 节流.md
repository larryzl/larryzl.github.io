---
layout: post
title: "Django restframework 框架笔记 (三) 节流"
date: 2019-04-26 16:58:16 +0800
category: Django
tags: [Django]
---
* content
{:toc}


# 1. 添加自定义节流配置

> 功能： 根据IP限制60秒内只能访问3次

## 1.1. 创建节流文件

在 `api/utils`下创建 `throttle.py`文件

```python
# utils/throttle.py

from rest_framework.throttling import BaseThrottle
import time

VISIT_RECORD = {}  # 保存访问记录


class VisitThrottle(BaseThrottle):
    """60s内只能访问3次"""

    def __init__(self):
        self.history = None  # 初始化访问记录

    def allow_request(self, request, view):
        # 获取用户ip (get_ident)
        remote_addr = self.get_ident(request)
        ctime = time.time()
        # 如果当前IP不在访问记录里面，就添加到记录
        if remote_addr not in VISIT_RECORD:
            VISIT_RECORD[remote_addr] = [ctime, ]  # 键值对的形式保存
            return True  # True表示可以访问
        # 获取当前ip的历史访问记录
        history = VISIT_RECORD.get(remote_addr)
        # 初始化访问记录
        self.history = history

        # 如果有历史访问记录，并且最早一次的访问记录离当前时间超过60s，就删除最早的那个访问记录，
        # 只要为True，就一直循环删除最早的一次访问记录
        while history and history[-1] < ctime - 60:
            history.pop()
        # 如果访问记录不超过三次，就把当前的访问记录插到第一个位置（pop删除最后一个）
        if len(history) < 3:
            history.insert(0, ctime)
            return True

    def wait(self):
        """还需要等多久才能访问"""
        ctime = time.time()
        return 60 - (ctime - self.history[-1])
```

## 1.2. 配置在全局配置文件中添加配置

`settings.py`

```python
REST_FRAMEWORK = {
    # 认证处理
    'DEFAULT_AUTHENTICATION_CLASSES': ['api.utils.auth.Authentication'],
    # 权限处理
    'DEFAULT_PERMISSION_CLASSES': ['api.utils.premission.VIPPremission'],
    # 节流处理
    'DEFAULT_THROTTLE_CLASSES': ['api.utils.throttle.VisitThrottle'],
}
```

## 1.3. 测试

![image-20200426172924503](/Users/lei/Library/Application Support/typora-user-images/image-20200426172924503.png)



# 2. 源码分析

## 2.1. 源码流程

1. `def dispatch(self, request, *args, **kwargs):`

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

2. `self.initial(request, *args, **kwargs)`

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
       self.check_permissions(request)
       # 节流配置
       self.check_throttles(request)
   ```

3. `self.check_throttles(request)`

   ```python
       def check_throttles(self, request):
           """
           Check if request should be throttled.
           Raises an appropriate exception if the request is throttled.
           """
           throttle_durations = []
           for throttle in self.get_throttles():
               if not throttle.allow_request(request, self):
                   throttle_durations.append(throttle.wait())
   
           if throttle_durations:
               # Filter out `None` values which may happen in case of config / rate
               # changes, see #1438
               durations = [
                   duration for duration in throttle_durations
                   if duration is not None
               ]
   
               duration = max(durations, default=None)
               self.throttled(request, duration)	
   ```

4. get_throttles(self):

   ```python
   def get_throttles(self):
       """
       Instantiates and returns the list of throttles that this view uses.
       """
       return [throttle() for throttle in self.throttle_classes]
   ```

5. `APIView(View)`

   ```python
   class APIView(View):
   
       # The following policies may be set at either globally, or per-view.
       renderer_classes = api_settings.DEFAULT_RENDERER_CLASSES
       parser_classes = api_settings.DEFAULT_PARSER_CLASSES
       authentication_classes = api_settings.DEFAULT_AUTHENTICATION_CLASSES
       # 节流配置
       throttle_classes = api_settings.DEFAULT_THROTTLE_CLASSES
       permission_classes = api_settings.DEFAULT_PERMISSION_CLASSES
       content_negotiation_class = api_settings.DEFAULT_CONTENT_NEGOTIATION_CLASS
       metadata_class = api_settings.DEFAULT_METADATA_CLASS
       versioning_class = api_settings.DEFAULT_VERSIONING_CLASS
   ```

6. drf 默认的节流配置为空 `rest_framework/settings.py` 

   ```python
   'DEFAULT_THROTTLE_CLASSES': [],
     
   'DEFAULT_THROTTLE_RATES': {
           'user': None,
           'anon': None,
    },
   ```

   

## 2.2. 内置节流类

内置节流类文件为 `rest_framework/throttling.py`

### 2.2.1. BaseThrottle

自己需要重写`allow_request`和`wait`方法，`allow_request` 默认抛出异常

`get_ident`就是获取ip

```python
class BaseThrottle:
    """
    Rate throttling of requests.
    """

    def allow_request(self, request, view):
        """
        Return `True` if the request should be allowed, `False` otherwise.
        """
        raise NotImplementedError('.allow_request() must be overridden')

    def get_ident(self, request):
        """
        Identify the machine making the request by parsing HTTP_X_FORWARDED_FOR
        if present and number of proxies is > 0. If not use all of
        HTTP_X_FORWARDED_FOR if it is available, if not use REMOTE_ADDR.
        """
        xff = request.META.get('HTTP_X_FORWARDED_FOR')
        remote_addr = request.META.get('REMOTE_ADDR')
        num_proxies = api_settings.NUM_PROXIES

        if num_proxies is not None:
            if num_proxies == 0 or xff is None:
                return remote_addr
            addrs = xff.split(',')
            client_addr = addrs[-min(num_proxies, len(addrs))]
            return client_addr.strip()

        return ''.join(xff.split()) if xff else remote_addr

    def wait(self):
        """
        Optionally, return a recommended number of seconds to wait before
        the next request.
        """
        return None
```

### 2.2.2. SimpleRateThrottle

```python
class SimpleRateThrottle(BaseThrottle):
    """
    A simple cache implementation, that only requires `.get_cache_key()`
    to be overridden.

    The rate (requests / seconds) is set by a `rate` attribute on the View
    class.  The attribute is a string of the form 'number_of_requests/period'.

    Period should be one of: ('s', 'sec', 'm', 'min', 'h', 'hour', 'd', 'day')

    Previous request information used for throttling is stored in the cache.
    """
    cache = default_cache
    timer = time.time
    cache_format = 'throttle_%(scope)s_%(ident)s'

    scope = None     # 该值需要自定义
    THROTTLE_RATES = api_settings.DEFAULT_THROTTLE_RATES

    def __init__(self):
        if not getattr(self, 'rate', None):
            self.rate = self.get_rate()
        self.num_requests, self.duration = self.parse_rate(self.rate)

    def get_cache_key(self, request, view):
        """
        Should return a unique cache-key which can be used for throttling.
        Must be overridden.

        May return `None` if the request should not be throttled.
        """
        raise NotImplementedError('.get_cache_key() must be overridden')

    def get_rate(self):
        """
        Determine the string representation of the allowed request rate.
        """
        if not getattr(self, 'scope', None):
            msg = ("You must set either `.scope` or `.rate` for '%s' throttle" %
                   self.__class__.__name__)
            raise ImproperlyConfigured(msg)

        try:
            return self.THROTTLE_RATES[self.scope]
        except KeyError:
            msg = "No default throttle rate set for '%s' scope" % self.scope
            raise ImproperlyConfigured(msg)

    def parse_rate(self, rate):
        """
        Given the request rate string, return a two tuple of:
        <allowed number of requests>, <period of time in seconds>
        """
        if rate is None:
            return (None, None)
        num, period = rate.split('/')
        num_requests = int(num)
        duration = {'s': 1, 'm': 60, 'h': 3600, 'd': 86400}[period[0]]
        return (num_requests, duration)

    def allow_request(self, request, view):
        """
        Implement the check to see if the request should be throttled.

        On success calls `throttle_success`.
        On failure calls `throttle_failure`.
        """
        if self.rate is None:
            return True

        self.key = self.get_cache_key(request, view)
        if self.key is None:
            return True

        self.history = self.cache.get(self.key, [])
        self.now = self.timer()

        # Drop any requests from the history which have now passed the
        # throttle duration
        while self.history and self.history[-1] <= self.now - self.duration:
            self.history.pop()
        if len(self.history) >= self.num_requests:
            return self.throttle_failure()
        return self.throttle_success()

    def throttle_success(self):
        """
        Inserts the current request's timestamp along with the key
        into the cache.
        """
        self.history.insert(0, self.now)
        self.cache.set(self.key, self.history, self.duration)
        return True

    def throttle_failure(self):
        """
        Called when a request to the API has failed due to throttling.
        """
        return False

    def wait(self):
        """
        Returns the recommended next request time in seconds.
        """
        if self.history:
            remaining_duration = self.duration - (self.now - self.history[-1])
        else:
            remaining_duration = self.duration

        available_requests = self.num_requests - len(self.history) + 1
        if available_requests <= 0:
            return None

        return remaining_duration / float(available_requests)
```

### 2.2.3. 通过继承 `SimpleRateThrottle` 来实现节流

1. 修改 `throttle.py` 文件

   ```python
   from rest_framework.throttling import SimpleRateThrottle
   
   
   class UserThrottle(SimpleRateThrottle):
       scope = "user"
   
       def get_cache_key(self, request, view):
           return request.user.username
   ```

2. 修改`settings.py`

   ```python
   REST_FRAMEWORK = {
       # 认证处理
       'DEFAULT_AUTHENTICATION_CLASSES': ['api.utils.auth.Authentication'],
       # 权限处理
       'DEFAULT_PERMISSION_CLASSES': ['api.utils.premission.VIPPremission'],
       # 节流处理
       'DEFAULT_THROTTLE_CLASSES': ['api.utils.throttle.UserThrottle'],
       # 具体节流配置 , 限制 每分钟只能访问3次
       'DEFAULT_THROTTLE_RATES': {
           'user': '3/m',
       },
   }
   ```

