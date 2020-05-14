---
layout: post
title: "Django restframework 框架笔记 (七) generic与mixins"
date: 2019-05-07 15:52:29 +0800
category: Django
tags: [Django]
---
* content
{:toc}
# 1. GenericAPIView 使用

> 在使用GenericAPIView中，需要注意定义的属性，分别是：
>
> - queryset         指定queryset
> - serializer_class    指定serializer
>
> - pagination_class   指定分页类
> - filter_backends    指定过滤类
>
> - lookup_field      查询单一数据库对象时使用的条件字段，默认为'pk'
> - lookup_url_kwarg   查询单一数据时URL中的参数关键字名称，默认与look_field相同
>
> 

## 1.1. url.py

```python
from django.urls import re_path
from . import views

urlpatterns = [
    re_path(r'^v1/book/$', views.BookAPIView.as_view()),  # 继承APIView

    re_path(r'^v2/book/$', views.BookGenericAPIView.as_view()),
    re_path(r'v2/book/(?P<pk>.*)/$', views.BookGenericAPIView.as_view())
]
```



## 1.2. views.py

```python
from rest_framework.generics import GenericAPIView
from rest_framework.views import APIView
from . import models, serializers
from utils.response import APIResponse


# Create your views here.

class BookAPIView(APIView):
    def get(self, request, *args, **kwargs):
        queryset = models.Book.objects.filter(is_active=True)
        ser = serializers.BookModelSerializer(instance=queryset, many=True)
        return APIResponse(results=ser.data)


"""
class GenericAPIView(views.APIView):
GenericAPIView 只继承 APIView ，在其内部实现了三个方法(其他内部方法是作为这三个方法的子方法使用)

1. 对多条数据 get_queryset

def get_queryset(self):
    # 判断是否有 queryset 这个类属性 ,默认为空，在继承后，需要赋值      
    assert self.queryset is not None, (
        "'%s' should either include a `queryset` attribute, "
        "or override the `get_queryset()` method."
        % self.__class__.__name__
    )
    # 类型判断，如果通过返回 queryset.all()
    queryset = self.queryset
    if isinstance(queryset, QuerySet):
        # Ensure queryset is re-evaluated on each request.
        queryset = queryset.all()
    return queryset
   

2. get_object()     # 对单独数据
def get_object(self):
    # 过滤queryset
    queryset = self.filter_queryset(self.get_queryset())

    # Perform the lookup filtering.
    lookup_url_kwarg = self.lookup_url_kwarg or self.lookup_field

    assert lookup_url_kwarg in self.kwargs, (
        'Expected view %s to be called with a URL keyword argument '
        'named "%s". Fix your URL conf, or set the `.lookup_field` '
        'attribute on the view correctly.' %
        (self.__class__.__name__, lookup_url_kwarg)
    )

    filter_kwargs = {self.lookup_field: self.kwargs[lookup_url_kwarg]}
    obj = get_object_or_404(queryset, **filter_kwargs)

    # May raise a permission denied
    self.check_object_permissions(self.request, obj)

    return obj
    
    
3. get_serializer()
# 返回 serializer_class(*args, **kwargs)
def get_serializer(self, *args, **kwargs):
    serializer_class = self.get_serializer_class()
    kwargs['context'] = self.get_serializer_context()
    return serializer_class(*args, **kwargs)
    

def get_serializer_class(self):
    # 判断是否指定 serializer_class
    assert self.serializer_class is not None, (
        "'%s' should either include a `serializer_class` attribute, "
        "or override the `get_serializer_class()` method."
        % self.__class__.__name__
    )
    return self.serializer_class

"""


class BookGenericAPIView(GenericAPIView):
    queryset = models.Book.objects.filter(is_active=True)
    serializer_class = serializers.BookModelSerializer
    # 指定索引字段,与 url.py 中的值对应
    lookup_field = 'pk'

    # 多查
    # def get(self, request, *args, **kwargs):
    #     queryset = self.get_queryset()
    #     ser = self.get_serializer(queryset, many=True)
    #     return APIResponse(results=ser.data)

    # 单查
    def get(self, request, *args, **kwargs):
        queryset = self.get_object()
        ser = self.get_serializer(queryset)
        return APIResponse(results=ser.data)


```

# 2. mixins 使用

> mixins中只有五个类，分别为：
>
> | mixins             | 作用                                 | 对应方法  |
> | ------------------ | ------------------------------------ | --------- |
> | ListModelMixin     | 定义list方法，返回一个queryset的列表 | GET       |
> | RetrieveModelMixin | 定义retrieve方法，返回一个具体实例   | GET       |
> | UpdateModelMixin   | 定义update方法，对某个实例进行更新   | PUT/PATCH |
> | DestroyModelMixin  | 定义delete方法，删除某个实例         | DELETE    |
> | CreateModelMixin   | 定义create方法，创建一个实例         | POST      |



## 2.1. mixins类介绍

### 2.1.1.  CreateModelMixin

```python
# 源码
class CreateModelMixin(object):
    """
    Create a model instance ==>创建一个实例
    """
    def create(self, request, *args, **kwargs):

    		# 1. 获取相关serializer
        serializer = self.get_serializer(data=request.data)
        # 2. 进行serializer的验证
        # 3. raise_exception=True,一旦验证不通过，不再往下执行，直接引发异常
        serializer.is_valid(raise_exception=True)
        # 调用perform_create()方法，保存实例
        self.perform_create(serializer)
        headers = self.get_success_headers(serializer.data)
        # 5. 返回 Response
        return Response(serializer.data, status=status.HTTP_201_CREATED, headers=headers)
    def perform_create(self, serializer):
    		# 4. 保存实例
        serializer.save()

    def get_success_headers(self, data):
        try:
            return {'Location': str(data[api_settings.URL_FIELD_NAME])}
        except (TypeError, KeyError):
            return {}
```

**流程：**

```
post -> 获取相关serializer -> 进行数据验证	(成功)-- > 保存实例 -> Response

																				(失败)|--->Exception->Response
```



### 2.1.2. ListModelMixin

```python
# 源码
class ListModelMixin(object):
    """
    List a queryset.==> 列表页获取
    """
    def list(self, request, *args, **kwargs):
        queryset = self.filter_queryset(self.get_queryset())

        # 这是一个分页功能，如果在viewset中设置了pagination_class，那么这里就会起作用
        # 获取当前页的queryset，如果不存在分页，返回None
        page = self.paginate_queryset(queryset)

        if page is not None:
        # 分页不为空，那么不能简单的执行Response(serializer.data)
        # 还需要将相关的page信息序列化在进行响应
            serializer = self.get_serializer(page, many=True)
            return self.get_paginated_response(serializer.data)

        serializer = self.get_serializer(queryset, many=True)
        return Response(serializer.data)
```

ListModelMixin一般用来获取列表页，大多数情况下比较简单，不需要重写相关的方法。

### 2.1.3. RetrieveModelMixin

```python
# 源码
class RetrieveModelMixin(object):
    """
    Retrieve a model instance.==> 获取某一个对象的具体信息
    """
    def retrieve(self, request, *args, **kwargs):
        # 一般访问的url都为/obj/id/这种新式
        # get_object()可以获取到这个id的对象
        # 注意在viewset中设置lookup_field获取重写get_object()方法可以指定id具体对象是什么~！
        instance = self.get_object()
        serializer = self.get_serializer(instance)
        return Response(serializer.data)
```

对retrieve这个方法的重写几率比较高，例如我们在增加点击数的时候，经常要对其进行一个重写。

### 2.1.4. UpdateModelMixin

```python
# 源码
class UpdateModelMixin(object):
    """
    Update a model instance.==> 更新某个具体对象的内容
    """
    def update(self, request, *args, **kwargs):
        partial = kwargs.pop('partial', False)
        instance = self.get_object()
        serializer = self.get_serializer(instance, data=request.data, partial=partial)
        serializer.is_valid(raise_exception=True)
        self.perform_update(serializer)

        if getattr(instance, '_prefetched_objects_cache', None):
            # If 'prefetch_related' has been applied to a queryset, we need to
            # forcibly invalidate the prefetch cache on the instance.
            instance._prefetched_objects_cache = {}

        return Response(serializer.data)

    def perform_update(self, serializer):
        serializer.save()
		# patch方法调用
    def partial_update(self, request, *args, **kwargs):
        kwargs['partial'] = True
        return self.update(request, *args, **kwargs)
```

RetrieveModelMixin的实现逻辑基本整合了Create以及Retrieve，先得到具体的实例，再对其进行验证以及保存，如果需要对更新这个逻辑进行自定义，那么需要重写perform_update( )方法，而尽量少去重写update( )

### 2.1.5. DestroyModelMixin

```python
# 源码
class DestroyModelMixin(object):
    """
    Destroy a model instance.
    """
    def destroy(self, request, *args, **kwargs):
        instance = self.get_object()
        self.perform_destroy(instance)
        return Response(status=status.HTTP_204_NO_CONTENT)

    def perform_destroy(self, instance):
        instance.delete()
```

DestroyModelMixin的逻辑也相对比较简单，我们取CreateModelMixin下面的例子，当我们取消收藏，那么我们的DestroyModelMixin就发挥作用了。同理

## 2.2. 举个栗子

`view.py`:

```python
class BookGenericAPIView(ListModelMixin, CreateModelMixin, UpdateModelMixin, RetrieveModelMixin, GenericAPIView):
    queryset = models.Book.objects.filter(is_active=True)
    serializer_class = serializers.BookModelSerializer
    # 指定索引字段,与 url.py 中的值对应
    lookup_field = 'pk'

    # def get(self, request, *args, **kwargs):
    #     return self.list(request, *args, **kwargs)

    # 调用ListModelMixin中的 list 方法
    def get(self, request, *args, **kwargs):
        if "pk" in kwargs:  # 单条查询
            return APIResponse(results=self.retrieve(request, *args, **kwargs).data)
        # 获取self.list() 返回的 response对象
        response = self.list(request, *args, **kwargs)
        return APIResponse(results=response.data)

    # 调用 CreateModelMixin 中的create方法
    def post(self, request, *args, **kwargs):
        response = self.create(request, *args, **kwargs)
        return APIResponse(results=response.data)

    def put(self, request, *args, **kwargs):
        response = self.update(request, *args, **kwargs)
        return APIResponse(results=response.data)

    def patch(self, request, *args, **kwargs):
        response = self.partial_update(request, *args, **kwargs)
        return APIResponse(results=response.data)
    
```



# 3. generic 其他类使用

在 `generic`模块中 对`mixins`进行了封装，在日常使用中只需要继承相关类即可

## 3.1. 类介绍

### 3.1.1. ListAPIView 

```python
class ListAPIView(mixins.ListModelMixin,
                  GenericAPIView):
    """
    Concrete view for listing a queryset.
    """
    def get(self, request, *args, **kwargs):
        return self.list(request, *args, **kwargs)

```

### 3.1.2.  ListCreateAPIView 

```python
class ListCreateAPIView(mixins.ListModelMixin,
                        mixins.CreateModelMixin,
                        GenericAPIView):
    """
    Concrete view for listing a queryset or creating a model instance.
    """
    def get(self, request, *args, **kwargs):
        return self.list(request, *args, **kwargs)

    def post(self, request, *args, **kwargs):
        return self.create(request, *args, **kwargs)

```

### 3.1.3. RetrieveAPIVew 

```python
class RetrieveAPIView(mixins.RetrieveModelMixin,
                      GenericAPIView):
    """
    Concrete view for retrieving a model instance.
    """
    def get(self, request, *args, **kwargs):
        return self.retrieve(request, *args, **kwargs)
```



### 3.1.4. RetrieveUpdateAPIView

```python
class RetrieveUpdateAPIView(mixins.RetrieveModelMixin,
                            mixins.UpdateModelMixin,
                            GenericAPIView):
    """
    Concrete view for retrieving, updating a model instance.
    """
    def get(self, request, *args, **kwargs):
        return self.retrieve(request, *args, **kwargs)

    def put(self, request, *args, **kwargs):
        return self.update(request, *args, **kwargs)

    def patch(self, request, *args, **kwargs):
        return self.partial_update(request, *args, **kwargs)
```

### 3.1.5. UpdateAPIView

```python
class UpdateAPIView(mixins.UpdateModelMixin,
                    GenericAPIView):
    """
    Concrete view for updating a model instance.
    """
    def put(self, request, *args, **kwargs):
        return self.update(request, *args, **kwargs)

    def patch(self, request, *args, **kwargs):
        return self.partial_update(request, *args, **kwargs)
```



## 3.2. 举个栗子

在继承`ListCreateAPIView` 类后不需要在写 `get()`或 `post()` 方法

继承`UpdateAPIView` 后实现了 `update()或 patch()`方法

```python
from rest_framework.generics import ListCreateAPIView, UpdateAPIView

class BookGenericAPIView(ListCreateAPIView, UpdateAPIView):
    queryset = models.Book.objects.filter(is_active=True)
    serializer_class = serializers.BookModelSerializer
    # 指定索引字段,与 url.py 中的值对应
    lookup_field = 'pk'
```

