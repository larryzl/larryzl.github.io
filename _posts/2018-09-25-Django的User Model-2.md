---
layout: post
title: "Django的User Model(2)"
category: Django
tags: [Django,Python]
---
* content
{:toc}



### django 重写用户模型

Django内建的User模型可能不适合某些类型的项目。例如，在某些网站上使用邮件地址而不是用户名作为身份的标识可能更合理。

#### 1. 修改配置文件，覆盖默认的User模型

Django允许你通过修改setting.py文件中的 `AUTH_USER_MODEL` 设置覆盖默认的User模型，其值引用一个自定义的模型。
	
	AUTH_USER_MODEL = 'myapp.MyUser'

上面的值表示Django应用的名称（必须位于INSTALLLED_APPS中）和你想使用的User模型的名称。

**注意:**
	
1.在创建任何迁移或者第一次运行 manager.py migrate 前设置 `AUTH_USER_MODEL`。  
	
设置`AUTH_USER_MODEL`对你的数据库结构有很大的影响。它改变了一些会使用到的表格，并且会影响到一些外键和多对多关系的构造。在你有表格被创建后更改此设置是不被 makemigrations 支持的，并且会导致你需要手动修改数据库结构，从旧用户表中导出数据，可能重新应用一些迁移。

**警告:**
1.确保 `AUTH_USER_MODEL` 引用的模型在所属app中第一个迁移文件中被创建  

由于Django的可交换模型的动态依赖特性的局限，你必须确保 `AUTH_USER_MODEL` 引用的模型在所属app中第一个迁移文件中被创建（通常命名为 0001_initial），否则你会碰到错误。


#### 2. 引用User模型

在`AUTH_USER_MODEL`设置为自定义用户模型时，如果你直接引用User（例如：通过一个外键引用它），你的代码将不能工作。你应该使用`django.contrib.auth.get_user_model()`来引用用户模型————指定的自定义用户模型或者User。

	from django.contrib.auth import get_user_model

	User = get_user_model()

当你定义一个外键或者到用户模型的多对多关系是，你应该使用`AUTH_USER_MODEL`设置来指定自定义的模型。

	from django.conf import settings
	from django.db import models
	
	class Article(models.Model):
   		author = models.ForeignKey(settings.AUTH_USER_MODEL)

#### 3. 指定自定义的用户模型

1. Django期望你自定义的User model 满足一些最低要求：

	1. 模型必须有一个唯一的字段可被用于识别目的。可以是一个用户名，电子邮件地址或任何其他独特属性。
	2. 定制一个User Model最简单的方式是构造一个兼容的用户模型继承于`AbstractBaseUser`.`AbstractBaseUser `提供了User类最核心的实现，包括哈希的passwords喝标识的密码重置。
2. 下面为一些`AbstractBaseUser `的子类必须定义关键的字段和方法：

	`USERNAME_FIELD`  
	必须设置。 设置认证标识，设置成标识的字段 `unique=True`

		class MyUser(AbstractBaseUser):
		    identifier = models.CharField(max_length=40, unique=True)
		    ...
		    USERNAME_FIELD = 'identifier'
	
	`REQUIRED_FIELDS`  
	必须设置。当通过createsuperuser管理命令创建一个用户时，用于提示的一个字段名称列表。

		class MyUser(AbstractBaseUser):
		    ...
		    date_of_birth = models.DateField()
		    height = models.FloatField()
		    ...
		    REQUIRED_FIELDS = ['date_of_birth', 'height']

	列表中不应该包含USERNAME_FIELD字段和password字段。
	
	|方法|说明|
	|---|---|
	|`is_active`|必须定义。 一个布尔属性，标识用户是否是 "active" 的。AbstractBaseUser默认为 Ture。|
	|`get_full_name()`|必须定义。 long格式的用户标识。|
	|`get_short_name()`| 必须定义。 short格式的用户标识。|

3. 下面为一些AbstractBaseUser的子类可以使用的方法：

	|类名|说明|
	|---|----|
	|`get_username()`|返回 `USERNAME_FIELD` 的值。|
	|`is_anonymous()`|一直返回 `False`。用来区分 `AnonymousUser`。|
	|`is_authenticated()`|一直返回 `Ture`。用来告诉用户已被认证。|
	|`set_password(raw_password)`|设置密码。按照给定的原始字符串设置用户的密码，taking care of the password hashing。 不保存 `AbstractBaseUser` 对象。如果没有给定密码，密码就会被设置成不使用，同用`set_unusable_password()`。|
	|`check_password(raw_password)`|检查密码是否正确。 给定的密码正确返回 `True`。|
	|`set_unusable_password()`|设置user无密码。 不同于密码为空，如果使用 `check_password()`，则不会返回`True`。不保存`AbstractBaseUser` 对象。|
	|`has_usable_password()`|如果设置了`set_unusable_password()`，返回`False`。|
	|`get_session_auth_hash()`|返回密码字段的HMAC。 Used for Session invalidation on password change.|
	

4. 为你的User模型自定义一个管理器

	如果你的`User`模型定义了这些字段：`username`, `email`, `is_staff`, `is_active`, `is_superuser`, `last_login`, and `date_joined`跟默认的`User`没什么区别, 那么你还不如仅仅替换Django的`UserManager`就行了; 总之,如果你的`User`定义了不同的字段, 你就要去自定义一个管理器，它继承自`BaseUserManager`并提供两个额外的方法:

	`create_user(username_field, password=None, other_fields)`  
	接受`username field`和`required`字段来创建用户。  
	例如，如果使用`email`作为`username field`， `date_of_birth`作为`required field`：

		def create_user(self, email, date_of_birth, password=None):
		    # create user here
		    ...

	`create_superuser(username_field, password, other_fields)`  
	接受username field和required字段来创建superuser。  
	例如，如果使用email作为username field， date_of_birth作为required field：
	
		def create_superuser(self, email, date_of_birth, password):
		    # create superuser here
		    ...
		    
	`create_superuser`中的password是必需的

#### 4. 扩展Django默认的User

如果你完全满意Django的用户模型和你只是想添加一些额外的属性信息,你只需继承`django.contrib.auth.models.AbstractUser` 然后添加自定义的属性。`AbstractUser` 作为一个抽象模型提供了默认的`User`的所有的实现（AbstractUser provides the full implementation of the default User as an abstract model.）。

#### 5. 自定义用户与内置身份验证表单

Django内置的forms和views和相关联的user model有一些先决条件。如果你的user model没有遵循同样的条件，则需要定义一个替代的form，通过form成为身份验证views配置的一部分。

|类名|说明|
|---|---|
|UserCreationForm|依赖于User Model. 扩展User时必须重写。|
|UserChangeForm|依赖于User Model. 扩展User时必须重写。|
|AuthenticationForm|Works with any subclass of AbstractBaseUser, and will adapt to use the field defined in USERNAME_FIELD.|
|PasswordResetForm|Assumes that the user model has a field named email that can be used to identify the user and a boolean field named is_active to prevent password resets for inactive users.|
|SetPasswordForm|Works with 任何AbstractBaseUser子类|
|PasswordChangeForm|Works with 任何AbstractBaseUser子类|
|AdminPasswordChangeForm|Works with 任何AbstractBaseUser子类|

#### 6. 自定义用户和django.contrib.admin

如果你想让你自定义的User模型也可以在站点管理上工作，那么你的模型应该再定义一些额外的属性和方法。 这些方法允许管理员去控制User到管理内容的访问:

|方法|说明|
|---|---|
|is_staff|是否允许user访问admin界面|
|is_active|用户是否活跃。|
|has_perm(perm, obj=None):|user是否拥有perm权限。|
|has_module_perms(app_label):|user是否拥有app中访问models的权限|

你同样也需要注册你自定义的用户模型到admin。  
	
如果你的自定义用户模型扩展于  	`django.contrib.auth.models.AbscustomauthtractUser`，你可以用django的 `django.contrib.auth.admin.UserAdmin `类。
	
如果你的用户模型扩展于 AbstractBaseUser，你需要自定义一个ModelAdmin类。
	
他可能继承于默认的`django.contrib.auth.admin.UserAdmin`。然而，你也需要覆写一些`django.contrib.auth.models.AbstractUs`er 字段的定义不在你自定义用户模型中的。


#### 7. 自定义用户和权限

如果想让在自定义用户模型中包含Django的权限控制框架变得简单，Django提供了`PermissionsMixin`。这是一个抽象的类，你可以为你的自定义用户模型中的类的层次结构中包含它。它提供给你所有Django权限类所必须的的方法和字段

1. 如果要定制User的权限系统，最简单的方法是继承PermissionsMixin

	源码：

		class PermissionsMixin(models.Model):
			"""
			A mixin class that adds the fields and methods necessary to support
			Django's Group and Permission model using the ModelBackend.
			"""
			is_superuser = models.BooleanField(_('superuser status'), default=False,
			    help_text=_('Designates that this user has all permissions without '
			                'explicitly assigning them.'))
			groups = models.ManyToManyField(Group, verbose_name=_('groups'),
			    blank=True, help_text=_('The groups this user belongs to. A user will '
			                            'get all permissions granted to each of '
			                            'their groups.'),
			    related_name="user_set", related_query_name="user")
			user_permissions = models.ManyToManyField(Permission,
			    verbose_name=_('user permissions'), blank=True,
			    help_text=_('Specific permissions for this user.'),
			    related_name="user_set", related_query_name="user")
			
			class Meta:
			    abstract = True
		
		def get_group_permissions(self, obj=None):
		    """
		    Returns a list of permission strings that this user has through their
		    groups. This method queries all available auth backends. If an object
		    is passed in, only permissions matching this object are returned.
		    """
		    permissions = set()
		    for backend in auth.get_backends():
		        if hasattr(backend, "get_group_permissions"):
		            permissions.update(backend.get_group_permissions(self, obj))
		    return permissions
		
		def get_all_permissions(self, obj=None):
		    return _user_get_all_permissions(self, obj)
		
		def has_perm(self, perm, obj=None):
		    """
		    Returns True if the user has the specified permission. This method
		    queries all available auth backends, but returns immediately if any
		    backend returns True. Thus, a user who has permission from a single
		    auth backend is assumed to have permission in general. If an object is
		    provided, permissions for this specific object are checked.
		    """
		
		    # Active superusers have all permissions.
		    if self.is_active and self.is_superuser:
		        return True
		
		    # Otherwise we need to check the backends.
		    return _user_has_perm(self, perm, obj)
		
		def has_perms(self, perm_list, obj=None):
		    """
		    Returns True if the user has each of the specified permissions. If
		    object is passed, it checks if the user has all required perms for this
		    object.
		    """
		    for perm in perm_list:
		        if not self.has_perm(perm, obj):
		            return False
		    return True
		
		def has_module_perms(self, app_label):
		    """
		    Returns True if the user has any permissions in the given app label.
		    Uses pretty much the same logic as has_perm, above.
		    """
		    # Active superusers have all permissions.
		    if self.is_active and self.is_superuser:
		        return True
		
		    return _user_has_module_perms(self, app_label)

2. Django内置的User对象就继承了AbstractBaseUser和PermissionsMixin：

	源码：

		class AbstractUser(AbstractBaseUser, PermissionsMixin):
		    """
		    An abstract base class implementing a fully featured User model with
		    admin-compliant permissions.
		    Username, password and email are required. Other fields are optional.
		    """
		    username = models.CharField(_('username'), max_length=30, unique=True,
		        help_text=_('Required. 30 characters or fewer. Letters, digits and '
		                    '@/./+/-/_ only.'),
		        validators=[
		            validators.RegexValidator(r'^[\w.@+-]+$',
		                                      _('Enter a valid username. '
		                                        'This value may contain only letters, numbers '
		                                        'and @/./+/-/_ characters.'), 'invalid'),
		        ],
		        error_messages={
		            'unique': _("A user with that username already exists."),
		        })
		    first_name = models.CharField(_('first name'), max_length=30, blank=True)
		    last_name = models.CharField(_('last name'), max_length=30, blank=True)
		    email = models.EmailField(_('email address'), blank=True)
		    is_staff = models.BooleanField(_('staff status'), default=False,
		        help_text=_('Designates whether the user can log into this admin '
		                    'site.'))
		    is_active = models.BooleanField(_('active'), default=True,
		        help_text=_('Designates whether this user should be treated as '
		                    'active. Unselect this instead of deleting accounts.'))
		    date_joined = models.DateTimeField(_('date joined'), default=timezone.now)
		
		    objects = UserManager()
		
		    USERNAME_FIELD = 'username'
		    REQUIRED_FIELDS = ['email']
		
		    class Meta:
		        verbose_name = _('user')
		        verbose_name_plural = _('users')
		        abstract = True
		
		    def get_full_name(self):
		        """
		        Returns the first_name plus the last_name, with a space in between.
		        """
		        full_name = '%s %s' % (self.first_name, self.last_name)
		        return full_name.strip()
		
		    def get_short_name(self):
		        "Returns the short name for the user."
		        return self.first_name
		
		    def email_user(self, subject, message, from_email=None, **kwargs):
		        """
		        Sends an email to this User.
		        """
		        send_mail(subject, message, from_email, [self.email], **kwargs)
		
		
		class User(AbstractUser):
		    """
		    Users within the Django authentication system are represented by this
		    model.
		    Username, password and email are required. Other fields are optional.
		    """
		    class Meta(AbstractUser.Meta):
		        swappable = 'AUTH_USER_MODEL'

3.  PermissionsMixin提供的这些方法和属性：

	
	|方法|说明|
	|---|---|	
	|is_superuser|布尔类型。 Designates that this user has all permissions without explicitly assigning them.|
	|get_group_permissions(obj=None)|Returns a set of permission strings that the user has, through their groups.If obj is passed in, only returns the group permissions for this specific object.|
	|get_all_permissions(obj=None)|Returns a set of permission strings that the user has, both through group and user permissions.If obj is passed in, only returns the permissions for this specific object.|
	|has_perm(perm, obj=None)|Returns True if the user has the specified permission, where perm is in the format "<app label>.<permission codename>" (see permissions). If the user is inactive, this method will always return False.If obj is passed in, this method won’t check for a permission for the model, but for this specific object.|
	|has_perms(perm_list, obj=None)|Returns True if the user has each of the specified permissions, where each perm is in the format "<app label>.<permission codename>". If the user is inactive, this method will always return False.<br>If obj is passed in, this method won’t check for permissions for the model, but for the specific object.|
	|has_module_perms(package_name)|Returns True if the user has any permissions in the given package (the Django app label). If the user is inactive, this method will always return False.|
	
	

#### 8. 官方提供的一个完整的例子

这是一个管理器允许的自定义user这个用户模型使用邮箱地址作为用户名，并且要求填写出生年月。it provides no permission checking, beyond a simple admin flag on the user account. This model would be compatible with all the built-in auth forms and views, except for the User creation forms. This example illustrates how most of the components work together, but is not intended to be copied directly into projects for production use.

	# models.py
	
	from django.db import models
	from django.contrib.auth.models import (
	    BaseUserManager, AbstractBaseUser
	)
	
	
	class MyUserManager(BaseUserManager):
	    def create_user(self, email, date_of_birth, password=None):
	        """
	        Creates and saves a User with the given email, date of
	        birth and password.
	        """
	        if not email:
	            raise ValueError('Users must have an email address')
	
	        user = self.model(
	            email=self.normalize_email(email),
	            date_of_birth=date_of_birth,
	        )
	
	        user.set_password(password)
	        user.save(using=self._db)
	        return user
	
	    def create_superuser(self, email, date_of_birth, password):
	        """
	        Creates and saves a superuser with the given email, date of
	        birth and password.
	        """
	        user = self.create_user(email,
	            password=password,
	            date_of_birth=date_of_birth
	        )
	        user.is_admin = True
	        user.save(using=self._db)
	        return user
	
	
	class MyUser(AbstractBaseUser):
	    email = models.EmailField(
	        verbose_name='email address',
	        max_length=255,
	        unique=True,
	    )
	    date_of_birth = models.DateField()
	    is_active = models.BooleanField(default=True)
	    is_admin = models.BooleanField(default=False)
	
	    objects = MyUserManager()
	
	    USERNAME_FIELD = 'email'
	    REQUIRED_FIELDS = ['date_of_birth']
	
	    def get_full_name(self):
	        # The user is identified by their email address
	        return self.email
	
	    def get_short_name(self):
	        # The user is identified by their email address
	        return self.email
	
	    def __str__(self):              # __unicode__ on Python 2
	        return self.email
	
	    def has_perm(self, perm, obj=None):
	        "Does the user have a specific permission?"
	        # Simplest possible answer: Yes, always
	        return True
	
	    def has_module_perms(self, app_label):
	        "Does the user have permissions to view the app `app_label`?"
	        # Simplest possible answer: Yes, always
	        return True
	
	    @property
	    def is_staff(self):
	        "Is the user a member of staff?"
	        # Simplest possible answer: All admins are staff
	        return self.is_admin


可以看到manager定义了`create_user()`和create_superuser()方法，MyUser定义了`USERNAME_FIELD`,`REQUIRED_FIELDS`字段和`get_full_name()`,`get_short_name()`方法，为了能与admin一起使用，还定义了`is_active`,`is_staff`,`has_perm()`,`has_module_perms()`

要在admin中注册自定义的MyUser，还需要在app的admin.py中重写UserCreationForm和UserChangeForm：

	# admin.py
	from django import forms
	from django.contrib import admin
	from django.contrib.auth.models import Group
	from django.contrib.auth.admin import UserAdmin
	from django.contrib.auth.forms import ReadOnlyPasswordHashField
	
	from customauth.models import MyUser
	
	
	class UserCreationForm(forms.ModelForm):
	    """A form for creating new users. Includes all the required
	    fields, plus a repeated password."""
	    password1 = forms.CharField(label='Password', widget=forms.PasswordInput)
	    password2 = forms.CharField(label='Password confirmation', widget=forms.PasswordInput)
	
	    class Meta:
	        model = MyUser
	        fields = ('email', 'date_of_birth')
	
	    def clean_password2(self):
	        # Check that the two password entries match
	        password1 = self.cleaned_data.get("password1")
	        password2 = self.cleaned_data.get("password2")
	        if password1 and password2 and password1 != password2:
	            raise forms.ValidationError("Passwords don't match")
	        return password2
	
	    def save(self, commit=True):
	        # Save the provided password in hashed format
	        user = super(UserCreationForm, self).save(commit=False)
	        user.set_password(self.cleaned_data["password1"])
	        if commit:
	            user.save()
	        return user
	
	
	class UserChangeForm(forms.ModelForm):
	    """A form for updating users. Includes all the fields on
	    the user, but replaces the password field with admin's
	    password hash display field.
	    """
	    password = ReadOnlyPasswordHashField()
	
	    class Meta:
	        model = MyUser
	        fields = ('email', 'password', 'date_of_birth', 'is_active', 'is_admin')
	
	    def clean_password(self):
	        # Regardless of what the user provides, return the initial value.
	        # This is done here, rather than on the field, because the
	        # field does not have access to the initial value
	        return self.initial["password"]
	
	
	class MyUserAdmin(UserAdmin):
	    # The forms to add and change user instances
	    form = UserChangeForm
	    add_form = UserCreationForm
	
	    # The fields to be used in displaying the User model.
	    # These override the definitions on the base UserAdmin
	    # that reference specific fields on auth.User.
	    list_display = ('email', 'date_of_birth', 'is_admin')
	    list_filter = ('is_admin',)
	    fieldsets = (
	        (None, {'fields': ('email', 'password')}),
	        ('Personal info', {'fields': ('date_of_birth',)}),
	        ('Permissions', {'fields': ('is_admin',)}),
	    )
	    # add_fieldsets is not a standard ModelAdmin attribute. UserAdmin
	    # overrides get_fieldsets to use this attribute when creating a user.
	    add_fieldsets = (
	        (None, {
	            'classes': ('wide',),
	            'fields': ('email', 'date_of_birth', 'password1', 'password2')}
	        ),
	    )
	    search_fields = ('email',)
	    ordering = ('email',)
	    filter_horizontal = ()
	
	# Now register the new UserAdmin...
	admin.site.register(MyUser, MyUserAdmin)
	# ... and, since we're not using Django's built-in permissions,
	# unregister the Group model from admin.
	admin.site.unregister(Group)



最后，别忘了在settings.py中定义AUTH_USER_MODEL:

	AUTH_USER_MODEL = 'customauth.MyUser'
