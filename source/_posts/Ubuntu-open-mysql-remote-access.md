---
title: Ubuntu14.04下如何开启Mysql远程访问
date: 2017-02-08 21:45:00
tags: [Ubuntu,MySQL]
categories: 数据库
description: "   "
---

在目录/etc/mysql下找到my.cnf，用vim编辑，找到my.cnf里面的
```java
#bind-address           = 127.0.0.1
```
将其只能本地ip访问的代码进行注释

然后用root登陆Mysql数据库
```java
im@58user:~$ mysql -u root -p
Enter password: 
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 6
Server version: 5.5.53-0ubuntu0.14.04.1 (Ubuntu)

Copyright (c) 2000, 2016, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> 
```
然后在执行
```java
mysql> grant all on *.* to root@'%' identified by '123';
myslq> flush privileges;
```
最后就可以在远程用刚才创建的用户和密码登陆mysql。

在执行对用户授权密码的时候可能会遇到下面的错误：
```java
ERROR 1290 (HY000): The MySQL server is running with the --skip-grant-tables option so it cannot execute this statement
```
解决方法：
>先刷新一下权限表。
mysql> flush privileges;
Query OK, 0 rows affected (0.01 sec)
>mysql> grant all on cactidb.* to dbuser@'localhost' identified by '123';
Query OK, 0 rows affected (0.00 sec)

想阅读作者的更多文章，可以查看我的公共号：
<center>![振兴书城](http://upload-images.jianshu.io/upload_images/1319879-612c4c66d40ce855.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</center>