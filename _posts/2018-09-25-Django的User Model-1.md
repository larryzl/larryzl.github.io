---
layout: post
title: "Django的User Model(1)"
category: Django
tags: [Python,Django]
---
* content
{:toc}



1. 确定 User Model

	我们推荐以下方式来确定某一个django项目使用的user model：

		# 使用默认User model时
		>>> from django.contrib.auth import get_user_model
		>>> get_user_model()
		<class 'django.contrib.auth.models.User'>
		
		# 使用自定义
		>>> from django.contrib.auth import get_user_model
		>>> get_user_model()
		<class 'xxx.models.UserProfile'>


2. 使用 `settings.AUTH_USER_MODEL`

	自从django 1.5之后，用户可以自定义User model了，如果需要外键使用user model ,官方推荐的方法如下：

	在settings中设置AUTH_USER_MODEL:

		# setting.py
		# 格式为 "<django_app名>.<model名>"
		AUTH_USER_MODEL = "myapp.NewUser"

	在models.py中使用

		# models.py
		from django.conf import settings
		from django.db import models
		
		class Article(models.Model):
			author = models.ForeignKey(settings.AUTH_USER_MODEL)
			title = models.CharField(max_length=255)

	还有需要注意的是，不要在外键中使用`get_user_model()`

3. 自定义 User Model

	**方法1： 扩展AbstractUser类**

	如果你对django自带的User model感到满意，又希望额外的field的话，你可以扩展AbstractUser类：

		# myapp/models.py
		from django.contrib.auth.models import AbstractUser
		from django.db import models
		
		class NewUser(AbstractUser):
			new_field = models.CharField(max_length=100)
			
	不要忘了在settings.py中设置：

		AUTH_USER_MODEL = "myapp.NewUser"
	
	**方法2： 扩展AbstractBaseUser类**

	AbstractBaseUser中含有3个field：password,last_login和is_active,如果你对django user model默认的first_name, last_name不满意, 或者只想保留默认的密码储存方式, 则可以选择这一方式.

	**方法3：使用OneToOneField**

	如果你想建立一个第三方模块发布在PyPi上，这一模块需要根据用户存储每个用户的额外信息. 或者我们的django项目中希望不同的用户拥有不同的field, 有些用户则需要不同field的组合, 且我们使用了方法1或方法2:

		# profiles/models.py
		from django.conf import settings
		from django.db import models
		
		from flavors.models import Flavor
		
		class EasterProfile(models.Model):
			user = models.OneToOneField(settings.AUTH_USER_MODEL)
			favorite_ice_cream = models.ForeignKey(Flavor,null=True,blank=True)
		
		class ScooperProfile(models.Model):
			user = models.OneToOneField(settings.AUTH_USER_MODEL)
        	scoops_scooped = models.IntergerField(default=0) 
        
	    class InventorProfile(models.Model):
	        user = models.OneToOneField(settings.AUTH_USER_MODEL)
	        flavors_invented = models.ManyToManyField(Flavor, null=True, blank=True)
		

	使用以上方法, 我们可以使用user.easterprofile.favorite_ice_cream获取相应的profile.

	使用这一方法的坏处可能就是增加了代码的复杂性.

