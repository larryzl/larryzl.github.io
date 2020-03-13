---
layout: post
title: "Dajngo 表关联 之 on_delete介绍"
date: 2018-04-16 15:02:00 +0800
category: Django
tags: [Django]
---
* content
{:toc}

## 一对多(ForeignKey)

	class ForeignKey(ForeignObject):
	    def __init__(self, to, on_delete, related_name=None, related_query_name=None,
	                 limit_choices_to=None, parent_link=False, to_field=None,
	                 db_constraint=True, **kwargs):
	        super().__init__(to, on_delete, from_fields=['self'], to_fields=[to_field], **kwargs)

## 一对一(OneToOneField)

	class OneToOneField(ForeignKey):
	    def __init__(self, to, on_delete, to_field=None, **kwargs):
	        kwargs['unique'] = True
	        super().__init__(to, on_delete, to_field=to_field, **kwargs)

从上面外键(`ForeignKey`)和一对一(`OneToOneField`)的参数中可以看出,都有`on_delete`参数,而 `django` 升级到2.0之后,表与表之间关联的时候,必须要写`on_delete`参数,否则会报异常:

	TypeError: __init__() missing 1 required positional argument: 'on_delete'

因此,整理一下`on_delete`参数的各个值的含义:

	on_delete=None,               # 删除关联表中的数据时,当前表与其关联的field的行为
	on_delete=models.CASCADE,     # 删除关联数据,与之关联也删除
	on_delete=models.DO_NOTHING,  # 删除关联数据,什么也不做
	on_delete=models.PROTECT,     # 删除关联数据,引发错误ProtectedError
	# models.ForeignKey('关联表', on_delete=models.SET_NULL, blank=True, null=True)
	on_delete=models.SET_NULL,    # 删除关联数据,与之关联的值设置为null（前提FK字段需要设置为可空,一对一同理）
	# models.ForeignKey('关联表', on_delete=models.SET_DEFAULT, default='默认值')
	on_delete=models.SET_DEFAULT, # 删除关联数据,与之关联的值设置为默认值（前提FK字段需要设置默认值,一对一同理）
	on_delete=models.SET,         # 删除关联数据,
 	 a. 与之关联的值设置为指定值,设置：models.SET(值)
	 b. 与之关联的值设置为可执行对象的返回值,设置：models.SET(可执行对象)

## 多对多(ManyToManyField)

	class ManyToManyField(RelatedField):
	    def __init__(self, to, related_name=None, related_query_name=None,
	                 limit_choices_to=None, symmetrical=None, through=None,
	                 through_fields=None, db_constraint=True, db_table=None,
	                 swappable=True, **kwargs):
	        super().__init__(**kwargs)

因为多对多(ManyToManyField)没有 on_delete 参数,所以略过不提.
来源：CSDN 
原文：https://blog.csdn.net/buxianghejiu/article/details/79086011 
版权声明：本文为博主原创文章，转载请附上博文链接！
