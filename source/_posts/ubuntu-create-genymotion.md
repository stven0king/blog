---
title: Ubuntu14.04创建Genymotion虚拟机
date: 2017-2-13 14:25:00
tags: [Ubuntu]
categories: Ubuntu
description: " "
---

最近工作开发环境有Windows切换到了Ubuntu，以前在Windows环境下使用Genymotion搞Android开发还蛮好用的。那么在Ubuntu环境下桌面创建Genymotion虚拟机呢，今天搞搞试试看～！～！

Virtualbox
------
先安装虚拟机软件Virtualbox，没有安装这个软件不能够使用Genymotion软件。
```java
sudo apt-get install virtualbox
```

Genymotion
-----

下载
====
先访问[Genymotion官网](https://www.genymotion.com/)，想要下载必须先注册Genymotion账号。

![genymotion.png](http://upload-images.jianshu.io/upload_images/1319879-4eab5cbad0aac5eb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

点击右上角的下载按钮，进入下载页面。Genymotion有好多版本，有些时收费的，作为开发者我们使用最基础的版本就够用的（PS:免费）。选择Get Genymotion personal version，进入personal Edit下载genymotion-2.8.1_x64.bin。

执行下边命令，生成名为genymotion的文件夹。
```java
chmod +x [InstallerPath]/genymotion.bin  
[InstallerPath]/genymotion.bin 
```

运行
====

进入genymotion文件夹后，我们可以看到名为genymotion的可运行程序，双击或者在命令行当中运行。

```java
im@58user:/usr/lib/x86_64-linux-gnu$ sudo /home/im/program/genymotion/./genymotion
Logging activities to file: /home/im/.Genymobile/genymotion.log
Logging activities to file: /home/im/.Genymobile/genymotion.log
Logging activities to file: /home/im/.Genymobile/Genymotion/deployed/Google Nexus 5X - 6.0.0 - API 23 - 1080x1920/genymotion-player.log
OpenGL connected to 192.168.56.101:25000
Port 22468 will be used for OpenGL data connections
```

如果没有问题那么则会像Windows环境下一样启动。

问题
--------
自古好事多磨

问题1：
===

```java
im@58user:/usr/lib/x86_64-linux-gnu$ sudo /home/im/program/genymotion/./genymotion
/home/im/program/genymotion/./genymotion: /usr/lib/x86_64-linux-gnu/libstdc++.so.6: version `CXXABI_1.3.8' not found (required by /home/im/program/genymotion/libQt5Core.so.5)
/home/im/program/genymotion/./genymotion: /usr/lib/x86_64-linux-gnu/libstdc++.so.6: version `GLIBCXX_3.4.20' not found (required by /home/im/program/genymotion/libQt5WebKit.so.5)
/home/im/program/genymotion/./genymotion: /usr/lib/x86_64-linux-gnu/libstdc++.so.6: version `CXXABI_1.3.8' not found (required by /home/im/program/genymotion/libicui18n.so.52)
/home/im/program/genymotion/./genymotion: /usr/lib/x86_64-linux-gnu/libstdc++.so.6: version `CXXABI_1.3.8' not found (required by /home/im/program/genymotion/libicuuc.so.52)
/home/im/program/genymotion/./genymotion: /usr/lib/x86_64-linux-gnu/libstdc++.so.6: version `GLIBCXX_3.4.20' not found (required by /home/im/program/genymotion/libQt5Qml.so.5)
```

执行genymotion程序时缺少相应的文件，上网找答案，提示更新gcc为4.9

下边为网络上的解决办法
```java
sudo add-apt-repository ppa:ubuntu-toolchain-r/test  
sudo apt-get update  
sudo apt-get install gcc-4.9 g++-4.9
```
然而在我的电脑环境中执行却没有办法更新gcc。

日志信息：
```java
im@58user:/usr/lib/x86_64-linux-gnu$ sudo apt-get install gcc-4.9 g++-4.9
[sudo] password for im: 
正在读取软件包列表... 完成
正在分析软件包的依赖关系树       
正在读取状态信息... 完成       
有一些软件包无法被安装。如果您用的是 unstable 发行版，这也许是
因为系统无法达到您要求的状态造成的。该版本中可能会有一些您需要的软件
包尚未被创建或是它们已被从新到(Incoming)目录移出。
下列信息可能会对解决问题有所帮助：

下列软件包有未满足的依赖关系：
 g++-4.9:i386 : 依赖: gcc-4.9-base:i386 (= 4.9.4-2ubuntu1~14.04.1) 但是 4.9.3-0ubuntu4 正要被安装
                依赖: libstdc++-4.9-dev:i386 (= 4.9.4-2ubuntu1~14.04.1) 但是它将不会被安装
                依赖: libcloog-isl4:i386 (>= 0.17) 但是它将不会被安装
                依赖: libmpc3:i386 但是它将不会被安装
                依赖: libmpfr4:i386 (>= 3.1.3) 但是它将不会被安装
 gcc-4.9:i386 : 依赖: cpp-4.9:i386 (= 4.9.4-2ubuntu1~14.04.1) 但是它将不会被安装
                依赖: gcc-4.9-base:i386 (= 4.9.4-2ubuntu1~14.04.1) 但是 4.9.3-0ubuntu4 正要被安装
                依赖: binutils:i386 (>= 2.24) 但是它将不会被安装
                依赖: libgcc-4.9-dev:i386 (= 4.9.4-2ubuntu1~14.04.1) 但是它将不会被安装
                依赖: libcloog-isl4:i386 (>= 0.17) 但是它将不会被安装
                依赖: libmpc3:i386 但是它将不会被安装
                依赖: libmpfr4:i386 (>= 3.1.3) 但是它将不会被安装
E: 无法修正错误，因为您要求某些软件包保持现状，就是它们破坏了软件包间的依赖关系。
```

好无奈，没有办法解决这个问题。

再才执行运行genymotion的命令
```java
im@58user:/usr/lib/x86_64-linux-gnu$ sudo /home/im/program/genymotion/./genymotion
```
查看输出的日志，有这么一段关键的信息 ``` /usr/lib/x86_64-linux-gnu/libstdc++.so.6: version xxx not found```  ，查看了一下该路径下的文件：
```java
im@58user:/usr/lib/x86_64-linux-gnu$ ls | grep "libstdc"
libstdc++.so.6
libstdc++.so.6.0.19
im@58user:/usr/lib/x86_64-linux-gnu$ pwd
/usr/lib/x86_64-linux-gnu
```
有libstdc++.so.6这个文件啊！！！

问题二：
要升级gcc（PS:升级失败），会不会gcc4.9比gcc4.8的libstdc++.so.6文件版本高。先下载libstdc++看看。
[http://ftp.debian.org/debian/pool/main/g/gcc-4.9/libstdc++6-4.9-dbg_4.9.2-10_amd64.deb](http://ftp.debian.org/debian/pool/main/g/gcc-4.9/libstdc++6-4.9-dbg_4.9.2-10_amd64.deb) 发现为.deb非常兴奋，是不是直接执行安装就行啦。结果依旧提示“依赖: gcc-4.9-base:i386 ”。

思考思考，先解压看看libstdc++6-4.9-dbg_4.9.2-10_amd64.deb文件里面都有什么：

![libstdc++6-4.9-dbg_4.9.2-10_amd64.deb.png](http://upload-images.jianshu.io/upload_images/1319879-7df3856a37aadea6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

找到libstdc++.so.6.0.20并提取出来并修改为libstdc++.so.6，再与 ```/usr/lib/x86_64-linux-gnu``` 目录下的libstdc++.so.6替换。再次运行genymotion，成功启动～！～！

解决一个问题的方法有好多种，多尝试，总能找到答案。

想阅读作者的更多文章，可以查看我的公共号：
<center>![振兴书城](http://upload-images.jianshu.io/upload_images/1319879-612c4c66d40ce855.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</center>