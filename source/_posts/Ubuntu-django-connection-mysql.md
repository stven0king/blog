---
title: Ubuntu 14.04下Django项目链接MySQL数据库
date: 2017-02-08 21:46:00
tags: [Ubuntu,MySQL]
categories: 数据库
description: " "
---

在成功安装MySQL-python-1.2.5后，开始配置django的mysql连接配置。
打开django项目的二级目录/Hello/Hello/setting.py文件。
默认情况下Django数据为sqlite：
```java
# Database
# https://docs.djangoproject.com/en/dev/ref/settings/#databases

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.sqlite3',
        'NAME': os.path.join(BASE_DIR, 'db.sqlite3'),
    }
}
```
现在我们将它修改为mysql数据库
```java
# Database
# https://docs.djangoproject.com/en/dev/ref/settings/#databases

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': 'mysite',	#数据库名称
        'USER': 'root',		#数据库的用户名
        'PASSWORD': '123',	#数据库对应用户的密码
        'HOST': '127.0.0.1',	#数据库主机
        'PORT': '3306',		#数据库默认端口号
    }
}
```

执行数据库同步脚本：
```java
python mange.py  syncdb
```
上面脚本可能在Django高版本执行报错，1.7及以上可以使用下边：
```java
python manage.py makemigrations
python manage.py migrate
```

执行结果
```java
im@58user:~/PythonProjects/Hello$ python manage.py migrate
System check identified some issues:
Operations to perform:
  Apply all migrations: admin, auth, contenttypes, sessions
Running migrations:
  Applying contenttypes.0001_initial... OK
  Applying auth.0001_initial... OK
  Applying admin.0001_initial... OK
  Applying admin.0002_logentry_remove_auto_add... OK
  Applying contenttypes.0002_remove_content_type_name... OK
  Applying auth.0002_alter_permission_name_max_length... OK
  Applying auth.0003_alter_user_email_max_length... OK
  Applying auth.0004_alter_user_username_opts... OK
  Applying auth.0005_alter_user_last_login_null... OK
  Applying auth.0006_require_contenttypes_0002... OK
  Applying auth.0007_alter_validators_add_error_messages... OK
  Applying auth.0008_alter_user_username_max_length... OK
  Applying sessions.0001_initial... OK

```

想阅读作者的更多文章，可以查看我的公共号：
<center>![振兴书城](http://upload-images.jianshu.io/upload_images/1319879-612c4c66d40ce855.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</center>