---
zlayout: post
title: "Django restframework 框架笔记 (八) viewset 视图集"
date: 2019-05-07 15:52:29 +0800
category: Django
tags: [Django]
---
* content
{:toc}
# 1. viewset介绍

DRF 允许将一组相关视图的逻辑组合到一个称为 `ViewSet` 的类中。

`ViewSet` 类只是一种基于类的 View，它不提供任何处理方法，如 `.get()` 或 `.post()`，而是提供诸如 `.list()` 和 `.create()` 之类的操作。

`ViewSet` 只在用 `.as_view()` 方法绑定到最终化视图时做一些相应操作。

通常，不是在 urlconf 中的视图集中明确注册视图，而是使用路由器类注册视图集，这会自动为您确定 urlconf。



## 1.1. 实例

继承ViewSet 后不需要在写 `get()` 方法

```python
from rest_framework import viewsets
from rest_framework.response import Response
from django.shortcuts import get_object_or_404
from rest_framework import status


class BookViewSet(viewsets.ViewSet):

    def list(self, request, *args, **kwargs):
        queryset = models.Book.objects.filter(is_active=True)
        ser = serializers.BookModelSerializer(instance=queryset, many=True)
        return Response({'code': 1000, 'msg': 'ok', 'results': ser.data}, status=status.HTTP_200_OK)

    def retrieve(self, request, *args, **kwargs):
        queryset = models.Book.objects.filter(is_active=True)
        book_obj = get_object_or_404(queryset, pk=kwargs.get("pk"))
        ser = serializers.BookModelSerializer(instance=book_obj)
        return Response({'code': 1000, 'msg': 'ok', 'results': ser.data}, status=status.HTTP_200_OK)

    def create(self, request, *args, **kwargs):
        ser = serializers.BookModelSerializer(data=request.data, many=False)
        ser.is_valid(raise_exception=False)
        obj = ser.save()
        return Response({'code': 1000, 'msg': 'ok', 'results': serializers.BookModelSerializer(instance=obj).data},
                        status=status.HTTP_200_OK)
```

继续配置`url.py` , 在 `as_view()` 中指定 `get/post/patch/put`等方法对应的方法

```python
    re_path(r'^v3/book/$', views.BookViewSet.as_view({'get': 'list', 'post': 'create'})),
    re_path(r'^v3/book/(?P<pk>.*)/$', views.BookViewSet.as_view({'get': 'retrieve'}))
```



# 2. ViewSetMixin 

Viewset的基类，它重写了原来 django view 中 `.as_view()` 方法，使得注册Url变得更加简单，原生 Django View 通过重写 get 和 post 方法的具体视图来达到实现逻辑
在 Viewset 中则可通过:

```python
view = MyViewSet.as_view({'get': 'list', 'post': 'create'})
```

在 `ViewSetMixin`类中 ，要求用户必须在`url`映射中指定 `actions` ，否则会抛出异常

```python
class ViewSetMixin:

    @classonlymethod
    def as_view(cls, actions=None, **initkwargs):
        cls.name = None
        cls.description = None
        cls.suffix = None
        cls.detail = None
        cls.basename = None
        # 如果 actions 为None ， 会抛出异常提示
        if not actions:
            raise TypeError("The `actions` argument must be provided when "
                            "calling `.as_view()` on a ViewSet. For example "
                            "`.as_view({'get': 'list'})`")

        for key in initkwargs:
            if key in cls.http_method_names:
                raise TypeError("You tried to pass in the %s method name as a "
                                "keyword argument to %s(). Don't do that."
                                % (key, cls.__name__))
            if not hasattr(cls, key):
                raise TypeError("%s() received an invalid keyword %r" % (
                    cls.__name__, key))

        if 'name' in initkwargs and 'suffix' in initkwargs:
            raise TypeError("%s() received both `name` and `suffix`, which are "
                            "mutually exclusive arguments." % (cls.__name__))

        def view(request, *args, **kwargs):
            self = cls(**initkwargs)
            self.action_map = actions
            for method, action in actions.items():
                handler = getattr(self, action)
                setattr(self, method, handler)

            if hasattr(self, 'get') and not hasattr(self, 'head'):
                self.head = self.get

            self.request = request
            self.args = args
            self.kwargs = kwargs

            # And continue as usual
            return self.dispatch(request, *args, **kwargs)

        # take name and docstring from class
        update_wrapper(view, cls, updated=())

        # and possible attributes set by decorators
        # like csrf_exempt from dispatch
        update_wrapper(view, cls.dispatch, assigned=())

        # We need to set these on the view function, so that breadcrumb
        # generation can pick out these bits of information from a
        # resolved URL.
        view.cls = cls
        view.initkwargs = initkwargs
        view.actions = actions
        return csrf_exempt(view)

    def initialize_request(self, request, *args, **kwargs):
        """
        Set the `.action` attribute on the view, depending on the request method.
        """
        request = super().initialize_request(request, *args, **kwargs)
        method = request.method.lower()
        if method == 'options':
            # This is a special case as we always provide handling for the
            # options method in the base `View` class.
            # Unlike the other explicitly defined actions, 'metadata' is implicit.
            self.action = 'metadata'
        else:
            self.action = self.action_map.get(method)
        return request

    def reverse_action(self, url_name, *args, **kwargs):
        """
        Reverse the action for the given `url_name`.
        """
        url_name = '%s-%s' % (self.basename, url_name)
        kwargs.setdefault('request', self.request)

        return reverse(url_name, *args, **kwargs)

    @classmethod
    def get_extra_actions(cls):
        """
        Get the methods that are marked as an extra ViewSet `@action`.
        """
        return [method for _, method in getmembers(cls, _is_extra_action)]

    def get_extra_action_url_map(self):
        """
        Build a map of {names: urls} for the extra actions.

        This method will noop if `detail` was not provided as a view initkwarg.
        """
        action_urls = OrderedDict()

        # exit early if `detail` has not been provided
        if self.detail is None:
            return action_urls

        # filter for the relevant extra actions
        actions = [
            action for action in self.get_extra_actions()
            if action.detail == self.detail
        ]

        for action in actions:
            try:
                url_name = '%s-%s' % (self.basename, action.url_name)
                url = reverse(url_name, self.args, self.kwargs, request=self.request)
                view = self.__class__(**action.kwargs)
                action_urls[view.get_view_name()] = url
            except NoReverseMatch:
                pass  # URL requires additional arguments, ignore

        return action_urls
```

viewset 中共包含 5 个类，其中 `ViewSetMixin（）` 为其他类的基类

## 2.1. viewsets 下面的视图类



### 2.1.1 ViewSet

```python
class ViewSet(ViewSetMixin, views.APIView):
    """
    The base ViewSet class does not provide any actions by default.
    """
    pass


```

继承 `ViewSetMixin, views.APIView` 两个类，只需要在url中配置映射关系，在视图类中编写具体方法，如：[1.1. 实例](#1.1. 实例)

### 2.1.2. GenericViewSet

```python
class GenericViewSet(ViewSetMixin, generics.GenericAPIView):
    """
    The GenericViewSet class does not provide any actions by default,
    but does include the base set of generic view behavior, such as
    the `get_object` and `get_queryset` methods.
    """
    pass

```

继承 `ViewSetMixin, generics.GenericAPIView` 两个类

使用如下:

```python
class BookViewSetMixin(GenericViewSet):
    serializer_class = serializers.BookModelSerializer
    queryset = models.Book.objects.all()

    def list(self, request, *args, **kwargs):
        queryset = models.Book.objects.filter(is_active=True)
        ser = serializers.BookModelSerializer(instance=queryset, many=True)
        return Response({'code': 1000, 'msg': 'ok', 'results': ser.data}, status=status.HTTP_200_OK)

    def create(self, request, *args, **kwargs):
        ser = serializers.BookModelSerializer(data=request.data, many=False)
        ser.is_valid(raise_exception=False)
        obj = ser.save()
        return Response({'code': 1000, 'msg': 'ok', 'results': serializers.BookModelSerializer(instance=obj).data},
                        status=status.HTTP_200_OK)

```



### 2.1.3. ReadOnlyModelViewSet

```python
class ReadOnlyModelViewSet(mixins.RetrieveModelMixin,
                           mixins.ListModelMixin,
                           GenericViewSet):
    """
    A viewset that provides default `list()` and `retrieve()` actions.
    """
    pass
```

继承 `mixins.RetrieveModelMixin,mixins.ListModelMixin,GenericViewSet` 三个类，在使用中，可以不用写`list()`方法

实例:

```python
from rest_framework.viewsets import ReadOnlyModelViewSet


class BookViewSetMixin(ReadOnlyModelViewSet):
    serializer_class = serializers.BookModelSerializer
    queryset = models.Book.objects.all()

    def retrieve(self, request, *args, **kwargs):
        queryset = models.Book.objects.filter(is_active=True)
        book_obj = get_object_or_404(queryset, pk=kwargs.get("pk"))
        ser = serializers.BookModelSerializer(instance=book_obj)
        return Response({'code': 1000, 'msg': 'ok', 'results': ser.data}, status=status.HTTP_200_OK)

    def create(self, request, *args, **kwargs):
        ser = serializers.BookModelSerializer(data=request.data, many=False)
        ser.is_valid(raise_exception=False)
        obj = ser.save()
        return Response({'code': 1000, 'msg': 'ok', 'results': serializers.BookModelSerializer(instance=obj).data},
                        status=status.HTTP_200_OK)

```



### 2.1.4. ModelViewSet

```python
class ModelViewSet(mixins.CreateModelMixin,
                   mixins.RetrieveModelMixin,
                   mixins.UpdateModelMixin,
                   mixins.DestroyModelMixin,
                   mixins.ListModelMixin,
                   GenericViewSet):
    """
    A viewset that provides default `create()`, `retrieve()`, `update()`,
    `partial_update()`, `destroy()` and `list()` actions.
    """
    pass

```

继承`ModelViewSet` 类后，内部已经实现了 create()`, `retrieve()`, `update()`,
    `partial_update()`, `destroy()` and `list()  这些方法，只需要 指定 `serializer_class` 和 `queryset` 字段就可以直接使用

实例：

```python
from rest_framework.viewsets import ModelViewSet


class BookViewSetMixin(ModelViewSet):
    serializer_class = serializers.BookModelSerializer
    queryset = models.Book.objects.all()
```



