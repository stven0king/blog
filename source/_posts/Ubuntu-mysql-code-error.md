---
title: Ubuntu的MySQL中文乱码问题--自己躺坑
date: 2017-02-08 21:59:00
tags: [MySQL,utf8]
categories: 数据库
description: "忽如一夜春风来，最近由Windows转到Ubuntu环境下学习了，在自学Django时链接MySQL数据库，遇到MySQL中文乱码这个问题折腾了我好长时间，终于搞好了，感觉自己把能躺的坑都躺过去了。"
---

最近一段时间学习Django，在进行与MySQL数据联合使用的插入数据的时候遇到下边的问题：
```java
/usr/local/lib/python2.7/dist-packages/Django-1.11.dev20170117002028-py2.7.egg/django/db/backends/mysql/base.py:109: Warning: Incorrect string value: '\xE6\x88\x90\xE5\x8A\x9F...' for column 'json' at row 1
  return self.cursor.execute(query, args)
[07/Feb/2017 12:15:21] "GET /index/ HTTP/1.1" 200 250
```

中文无法插入MySQL数据库～！～！

查看数据库编码
---
```java
mysql> show create database bangjob;
+----------+--------------------------------------------------------------------+
| Database | Create Database                                                    |
+----------+--------------------------------------------------------------------+
| bangjob  | CREATE DATABASE `bangjob` /*!40100 DEFAULT CHARACTER SET latin1 */ |
+----------+--------------------------------------------------------------------+
1 row in set (0.00 sec)
```

```java
mysql> show variables like'%char%';
+--------------------------+----------------------------+
| Variable_name            | Value                      |
+--------------------------+----------------------------+
| character_set_client     | utf8                       |
| character_set_connection | utf8                       |
| character_set_database   | latin1                     |
| character_set_filesystem | binary                     |
| character_set_results    | utf8                       |
| character_set_server     | latin1                     |
| character_set_system     | utf8                       |
| character_sets_dir       | /usr/share/mysql/charsets/ |
+--------------------------+----------------------------+
8 rows in set (0.00 sec)
```
修改数据库编码
---
```java
mysql> set character_set_database=utf8;
Query OK, 0 rows affected (0.00 sec)

mysql> set character_set_server=utf8;
Query OK, 0 rows affected (0.00 sec)
```
查看修改后的结果
---
```java
mysql> show variables like'%char%';
+--------------------------+----------------------------+
| Variable_name            | Value                      |
+--------------------------+----------------------------+
| character_set_client     | utf8                       |
| character_set_connection | utf8                       |
| character_set_database   | utf8                       |
| character_set_filesystem | binary                     |
| character_set_results    | utf8                       |
| character_set_server     | utf8                       |
| character_set_system     | utf8                       |
| character_sets_dir       | /usr/share/mysql/charsets/ |
+--------------------------+----------------------------+
8 rows in set (0.00 sec)
```

这时候继续插入就没有任何问题了。
如果真的是这样简单就好了，因为这样的修改作者在重启MySQL的后**设置失效！！！**

继续寻找其它方法
====

```java
sudo gedit /etc/mysql/my.cnf
```
在my.cnf文件的对应节点添加一下信息：
```java
[client]
default-character-set=utf8
[mysqld]
default-character-set=utf8
[mysql]
default-character-set=utf8
```
然后重启MySQL：
```java
/etc/init.d/mysql start
```

如果能重启那么再次查看数据库编码：
```java
mysql> show variables like "%char%";
+--------------------------+----------------------------+
| Variable_name            | Value                      |
+--------------------------+----------------------------+
| character_set_client     | utf8                       |
| character_set_connection | utf8                       |
| character_set_database   | utf8                       |
| character_set_filesystem | binary                     |
| character_set_results    | utf8                       |
| character_set_server     | utf8                       |
| character_set_system     | utf8                       |
| character_sets_dir       | /usr/share/mysql/charsets/ |
+--------------------------+----------------------------+
8 rows in set (0.01 sec)
```

如果真的时这样就好了，事情的发生总是不像想象的那么简单：
在重启MySQL服务的时候发现一直处于等待状态（PS：猜测发生了死锁什么的），这个时候执行 ：

```java
mysql -u root -p
 
```
则会抛出异常：

```java
ERROR 2002 (HY000): Can't connect to local MySQL server through socket '/var/run/mysqld/mysqld.sock' (2)
```

我就是想修改一下编码而已，为什么这么苦->->->->->!!!

解决这个问题的时候试过好多方法（重启，恢复）。。。。。
```java
 sudo /etc/init.d/mysql status
```
查看mysql的状态：```mysql respawn/post-start, (post-start) process 55665```

这些方式不能解决问题，还是从日志开始吧...
找到日至文件 /var/log/mysql/error.log 
![某一段日志内容.png](http://upload-images.jianshu.io/upload_images/1319879-1d95c8b3f32bb2c5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

继续找```ERROR 2002 (HY000): Can't connect to local MySQL server through socket '/var/run/mysqld/mysqld.sock' (2)```的解决办法。

答案：
***
[ mysqld  ]  下的 default-character-set=utf8'  改成
character_set_server=utf8
***

好了，终于可以重启MySQL了，并且重启后设置的编码依旧生效。

当然之前创建的数据库需要重新创建T_T
======
因为 ```show create database bangjob;```展示之前创建的数据编码依旧是**latin1**。。。。。

想阅读作者的更多文章，可以查看我的公共号：
<center>![振兴书城](http://upload-images.jianshu.io/upload_images/1319879-612c4c66d40ce855.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</center>