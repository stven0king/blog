---
title: Ubuntu14.04反编译Apk[试试就知道]
date: 2017-02-08 21:42:00
tags: [反编译,调试]
categories: [Android, Ubuntu]
description: "作为Android开发者反编译apk是我们需要掌握的技能，那么在Ubuntu环境下反编译怎么进行Apk的反编译呢"
---

作为Android开发者反编译apk是我们需要掌握的技能，那么在Ubuntu环境下反编译怎么进行Apk的反编译呢？

工具
---
- [dex2jar](https://sourceforge.net/projects/dex2jar/)
- [jd-gui](http://jd.benow.ca/jd-gui/downloads/jd-gui-0.3.5.linux.i686.tar.gz)

dex2jar使用
---
- 先参照上边提供的[地址]((https://sourceforge.net/projects/dex2jar/))下载并解压dex2jar
- 然后再使用unzip命令解压apk，我们会在目录下边看到.dex文件
- 执行反编译命令
```sh d2j-dex2jar.sh /home/im/Desktop/dex2jar/-debug-apk/classes.dex```

上述命令执行的过程中可能会遇到一些问题：
> 问题1：提示：```d2j-dex2jar.sh: 36: d2j-dex2jar.sh: ./d2j_invoke.sh: Permission denied```
原因：d2j_invoke.sh文件没有执行权限
解决：添加可执行权限：```sudo chmod +x d2j_invoke.sh```

> 问题2：生产的jar可能为空
原因：d2j-dex2jar.sh执行会依赖其它的脚本（单独拷贝出来执行会有问题）
解决：执行它的时候dex2jar的其它文件最好也在相同的目录

正确运行结果：
```java
im@58user:~/Downloads/dex2jar-2.0$ sudo chmod +x d2j_invoke.sh
im@58user:~/Downloads/dex2jar-2.0$ sh d2j-dex2jar.sh /home/im/Desktop/dex2jar/bangjob-apk/classes.dex
dex2jar /home/im/Desktop/dex2jar/bangjob-apk/classes.dex -> ./classes-dex2jar.jar
```
然后会在该目录生成classes-dex2jar.jar文件。


jd-gui使用
---
- 先参照上边提供的[地址](http://jd.benow.ca/jd-gui/downloads/jd-gui-0.3.5.linux.i686.tar.gz)下载文件
- 然后直接打开jd-gui

可能遇到的问题：
jd-gui程序执行的时候可能没有任何反应，那是因为操作系统可能缺少某些环境。执行该命令：```sudo apt-get install gtk2-engines-murrine:i386 libgtk2.0-0:i386 libcanberra-gtk-module:i386 libgtk2.0-0:i386 libxxf86vm1:i386 libsm6:i386 lib32stdc++6 lib32ncurses5 lib32bz2-1.0 libgtk2.0-0:i386 libxxf86vm1:i386 libsm-dev:i386 libcanberra-gtk3-module:i386```后然再运行jd-gui程序，画面即将展现～！～！

![dex2jar.png](http://upload-images.jianshu.io/upload_images/1319879-c8cc40591a38075b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

想阅读作者的更多文章，可以查看我的公共号：
<center>![振兴书城](http://upload-images.jianshu.io/upload_images/1319879-612c4c66d40ce855.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</center>