---
layout: post
title: "Django restframework 框架笔记 (六) 序列化"
date: 2019-04-27 14:10:48 +0800
category: Django
tags: [Django]
---
* content
{:toc}




# 1. 基础序列化使用(serializers.Serializer)

## 1.1. 目录层级:

```shell
app
├── api
│   ├── __init__.py
|   ├── admin.py
|   ├── apps.py
|   ├── models.py
|   ├── serializers.py
|   ├── tests.py
|   ├── urls.py
|   └── views.py
|       
├── app
│   ├── __init__.py
│   ├── asgi.py
│   ├── settings.py
│   ├── urls.py
│   └── wsgi.py
```



## 1.2. urls.py

```python
from django.urls import re_path

from . import views

urlpatterns = [
    re_path(r'^user/$', views.User.as_view(), name='user_list'),
    re_path(r'^user/(?P<pk>[0-9]+)/$', views.User.as_view(), name='user_list'),
]
```

## 1.3. models.py

```python
from django.db import models


class UserInfo(models.Model):
    USER_TYPE = (
        (1, '普通用户'),
        (2, 'VIP'),
        (3, 'SVIP')
    )

    username = models.CharField(max_length=32, unique=True, null=False)
    password = models.CharField(max_length=64, null=False)
    user_type = models.IntegerField(choices=USER_TYPE, default=1)
    avatar = models.ImageField(upload_to='avatar', default='avatar/default.png')

    class Meta:
        # db_table = "user_info"
        verbose_name = "用户"
        verbose_name_plural = verbose_name

    def __str__(self):
        return self.username
```

## 1.4. settings.py

在settings.py 中添加下面代码，并在项目目录中创建 `media`目录

```python
MEDIA_URL = '/media/'
MEDIA_DIR = os.path.join(BASE_DIR, 'media')
```

## 1.5. serializers.py

```python
from rest_framework import serializers
from django.conf import settings
from rest_framework import exceptions
from api import models
import re


# 序列化
class UserSerializers(serializers.Serializer):
  	# 需要序列化的字段
    username = serializers.CharField()
    user_type = serializers.SerializerMethodField()
    avatar = serializers.SerializerMethodField()
		
    def get_user_type(self, obj):
        return obj.get_user_type_display()

    def get_avatar(self, obj):
        url = "http://127.0.0.1:8000"
        return "{}{}{}".format(url, settings.MEDIA_URL, str(obj.avatar))


# 反序列化
class UserDeserializers(serializers.Serializer):
    # 需要处理的字段
    username = serializers.CharField(
        max_length=16,
        min_length=6,
        error_messages={
            'max_length': "超过最大长度"
        }
    )
    # 如果名称与models中不一致，可以通过 source=指定 
    pwd = serializers.CharField(source="password", max_length=32, min_length=5)
    re_pwd = serializers.CharField()
    user_type = serializers.CharField()

    # 局部验证
    def validate_username(self, value):
        if models.UserInfo.objects.filter(username=value):
            raise exceptions.ValidationError("用户名已存在")
        return value

    # 局部验证
    def validate_user_type(self, value):
        print(type(value))
        if int(value) not in dict(models.UserInfo.USER_TYPE).keys():
            raise exceptions.ValidationError("用户组编号错误")
        return value

    # 局部验证
    def validate_pwd(self, value):
        if len(re.findall(r'[a-z]+', value)) == 0:
            raise exceptions.ValidationError("密码中必须包含小写字母")
        if len(re.findall(r'[A-Z]+', value)) == 0:
            raise exceptions.ValidationError("密码中必须包含大写字母")
        return value

    # 全局验证
    def validate(self, attrs):
        pwd = attrs.get('password')
        re_pwd = attrs.pop('re_pwd')
        if pwd != re_pwd:
            raise exceptions.ValidationError({'pwd': "密码不匹配"})
        return attrs
		
    # 创建数据必须重写 create 方法，因为源码中的create方法直接抛出异常
    def create(self, validated_data):
        return models.UserInfo.objects.create(**validated_data)
```



# 2. 高级序列化使用(serializers.ModelSerializer)

## 2.1. urls

```python
from django.urls import re_path
from . import views

urlpatterns = [
    re_path(r'^author/$', views.AuthorView.as_view(), name='book_list'),
    re_path(r'^book/$', views.BookView.as_view(), name='book_list'),
    re_path(r'book/(?P<pk>.*)/$', views.BookView.as_view(), name='book_detail')
]
```



## 2.2. models.py

```python
from django.db import models

# Create your models here.

"""
Book    
Author
AuthorDetail
Publish
"""

class BaseModel(models.Model):
    is_active = models.BooleanField(default=True)
    create_time = models.DateTimeField(auto_now_add=True)

    class Meta:
        abstract = True

class Book(BaseModel):
    name = models.CharField(max_length=64, verbose_name="书名")
    price = models.DecimalField(max_digits=5, decimal_places=2, verbose_name="价格")
    img = models.ImageField(upload_to='img', default='img/default.jpg', verbose_name='封面图')
    publish = models.ForeignKey(to='Publish', on_delete=models.DO_NOTHING, verbose_name='出版社')
    authors = models.ManyToManyField(to='Author', verbose_name='作者')

    @property
    def publish_name(self):
        return self.publish.name

    @property
    def author_list(self):
        temp_list = []
        for author in self.authors.all():
            temp_list.append({
                "name": author.name,
                "age": author.detail.age,
                "gender": author.detail.gender,

            })
        return temp_list
        # return self.authors.values("name", "detail__age", "detail__gender").all()
        # return temp_list

    class Meta:
        db_table = "book"
        verbose_name = "书籍"
        verbose_name_plural = verbose_name

    def __str__(self):
        return self.name


class Author(BaseModel):
    name = models.CharField(max_length=64, verbose_name="姓名")

    class Meta:
        db_table = "author"
        verbose_name = "作者"
        verbose_name_plural = verbose_name

    def __str__(self):
        return self.name


class AuthorDetai(BaseModel):
    GENDER_CHOICE = (
        (0, '男'),
        (1, '女')

    )
    age = models.IntegerField(verbose_name="年龄")
    gender = models.IntegerField(choices=GENDER_CHOICE, default=0)
    author = models.OneToOneField(
        to='Author',
        on_delete=models.CASCADE,
        db_constraint=False,
        related_name='detail'  # 反向查询名称
    )

    class Meta:
        db_table = "author_detail"
        verbose_name = "作者详情"
        verbose_name_plural = verbose_name

    def __str__(self):
        return '%s的详情' % self.author.name


class Publish(BaseModel):
    name = models.CharField(max_length=32, verbose_name="名称")

    class Meta:
        db_table = "publish"
        verbose_name = "出版社"
        verbose_name_plural = verbose_name

    def __str__(self):
        return self.name

```

## 2.3. views

```python
from rest_framework import status
from rest_framework.views import APIView

from utils import codes
from utils.response import APIResponse
from . import models
from . import serializers


# Create your views here.
class AuthorView(APIView):
    def get(self, request, *args, **kwargs):
        pk = kwargs.get('pk')
        # 单挑数据
        if pk:
            try:
                obj = models.Author.objects.get(pk=pk, is_active=True)
                ser = serializers.AuthorModelSerializer(instance=obj, many=False)
            except:
                return APIResponse(data_status=codes.CODE_DATA_NOT_FOUND_ERROR, data_msg=codes.MSG_DATA_NOT_FOUND_ERROR)
        # 多条数据
        else:
            obj_queryset = models.Author.objects.filter(is_active=True)
            ser = serializers.AuthorModelSerializer(instance=obj_queryset, many=True)
        return APIResponse(results=ser.data)


class BookView(APIView):
    # 查询数据
    def get(self, request, *args, **kwargs):
        pk = kwargs.get('pk')
        # 单挑数据
        if pk:
            try:
                obj = models.Book.objects.get(pk=pk, is_active=True)
                ser = serializers.BookModelSerializer(instance=obj, many=False)
            except:
                return APIResponse(data_status=codes.CODE_DATA_NOT_FOUND_ERROR, data_msg=codes.MSG_DATA_NOT_FOUND_ERROR)
        # 多条数据
        else:
            obj_queryset = models.Book.objects.filter(is_active=True)
            ser = serializers.BookModelSerializer(instance=obj_queryset, many=True)
        return APIResponse(results=ser.data)

    # 添加数据
    def post(self, request, *args, **kwargs):
        # 单挑数据通过 {...} 字典方式添加，多条数据通过[{...},{...}] 列表方式添加
        if isinstance(request.data, list):
            many = True
        elif isinstance(request.data, dict):
            many = False
        else:
            return APIResponse(codes.CODE_PARAMETER_ERROR, codes.MSG_PARAMETER_ERROR)

        ser = serializers.BookModelSerializer(data=request.data, many=many)
        if ser.is_valid():
            ser_result = ser.save()
            return APIResponse(results=serializers.BookModelSerializer(instance=ser_result, many=many).data,
                               data_status=status.HTTP_201_CREATED)
        return APIResponse(codes.CODE_NOT_FOUND_ERROR, ser.errors)

    # 删除数据(将对应数据的 is_active 改为 False
    def delete(self, request, *args, **kwargs):
        pk = kwargs.get('pk')
        if pk:
            pks = [pk]
        else:
            pks = request.data.get('pks')
        if len(pks) < 1:
            return APIResponse(codes.CODE_DATA_ERROR, codes.MSG_DATA_ERROR)
        res = models.Book.objects.filter(pk__in=pks).update(is_active=False)
        if res > 0:
            return APIResponse(results={"lines": res})
        else:
            return APIResponse(codes.CODE_DATA_NOT_FOUND_ERROR, codes.MSG_DATA_NOT_FOUND_ERROR)

    def put(self, request, *args, **kwargs):
        """
        整体修改，接受PUT请求
        :param request:
        :param args:
        :param kwargs:
        :return:
        """

        request_data = []
        # 数据处理,将单改，多改数据格式化为 pk_list = [],request_data = []
        if isinstance(request.data, dict) and "pk" in kwargs:
            # 单数据修改
            pk_list = [kwargs.get('pk'), ]
            request_data = [request.data, ]
        elif isinstance(request.data, list) and "pk" not in kwargs:
            pk_list = []
            # 多数据修改 [ {pk:xx,name:xxx...},{}]
            for _data in request.data:
                if not isinstance(_data, dict) or "pk" not in _data:
                    return APIResponse(codes.CODE_DATA_ERROR, codes.MSG_DATA_ERROR)
                pk = _data.pop("pk")
                pk_list.append(pk)
                request_data = request.data
        else:
            # 数据有误
            return APIResponse(codes.CODE_DATA_ERROR, codes.MSG_DATA_ERROR)

        objs = models.Book.objects.filter(pk__in=pk_list, is_active=True)
        ser = serializers.BookModelSerializer(instance=objs, data=request_data, many=True)

        if ser.is_valid():
            ser.save()
            return APIResponse(results=serializers.BookModelSerializer(instance=objs, many=True).data,
                               data_status=status.HTTP_201_CREATED)
        else:
            return APIResponse(data_status=codes.CODE_DATA_NOT_FOUND_ERROR, data_msg=ser.errors)

    def patch(self, request, *args, **kwargs):
        """
        局部修改 ， 接受 patch 请求
        :param request:
        :param args:
        :param kwargs:
        :return:
        """
        pk = kwargs.get("pk")
        # pk = request.data.get('pk')
        obj = models.Book.objects.filter(pk=pk, is_active=True).first()
        ser = serializers.BookModelSerializer(instance=obj, data=request.data, partial=True)  # 局部更新 partial = True
        ser.is_valid(raise_exception=True)
        book_obj = ser.save()
        return APIResponse(data_status=codes.CODE_SUCCESS,
                           data_msg=serializers.BookModelSerializer(instance=book_obj).data)

```

## 2.4. serializers.py

```python
from rest_framework import serializers
from rest_framework.exceptions import ValidationError
from . import models


class PublishModelSerializer(serializers.ModelSerializer):
    class Meta:
        model = models.Publish
        fields = ('name',)


class AuthorModelSerializer(serializers.ModelSerializer):
    class Meta:
        model = models.Author
        fields = ('pk', 'name',)

        extra_kwargs = {
            'pk': {
                'read_only': True
            }
        }


class BookListSerializer(serializers.ListSerializer):
    # 重新update方法
    def update(self, instance, validated_data):
        for index, obj in enumerate(instance):
            self.child.update(obj, validated_data[index])

        return instance


class BookModelSerializer(serializers.ModelSerializer):
    class Meta:
        model = models.Book
        fields = ('pk', 'name', 'price', 'img', 'publish_name', 'author_list', 'authors', 'publish')
        extra_kwargs = {
            'pk': {
                'read_only': True  # 只有序列化中使用 read_only
            },
            'name': {
                'required': True,
                'max_length': 20,
                'min_length': 1,
                'error_messages': {
                    'require': '书籍名称不能为空'
                }
            },
            'authors': {
                'write_only': True,  # 只有在反序列化中使用 write_only
            },
            'publish': {
                'write_only': True
            },
            'img': {
                'read_only': True
            },
            'publish_name': {
                'read_only': True
            },

        }
        # 指定list serializer 类
        list_serializer_class = BookListSerializer

    # 局部验证
    def validate_name(self, value):
        if '红楼梦' in value:
            raise ValidationError('书籍名称有违法字符')
        return value

    # 全局验证
    def validate(self, attrs):
        if 'name' in attrs and 'authors' in attrs:
            if models.Book.objects.filter(name=attrs['name'], authors__in=attrs['authors']):
                raise ValidationError('书籍已存在')
        return attrs

```

## 2.5. 封装Response

```python
from rest_framework.response import Response
from rest_framework import status
from . import codes


class APIResponse(Response):
    def __init__(self, data_status=codes.CODE_SUCCESS, data_msg=codes.MSG_SUCCESS, results=None,
                 http_status=status.HTTP_200_OK,
                 headers=None, exception=False,
                 **kwargs):
        data = {
            'status': data_status,
            'msg': data_msg
        }

        # 如果又返回结果就将返回结果赋给到data中
        if results is not None:
            data["results"] = results

        # 如果传递其他的参数，将会被放到kwargs中被接收
        # if kwargs is not None:
        #     for k, v in kwargs.items():
        #         # 采用反射的方法，去赋值
        #         setattr(data, k, v)  # data[k] = v
        data.update(kwargs)
        super().__init__(data=data, status=http_status, headers=headers, exception=exception)

```



