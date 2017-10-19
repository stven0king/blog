---
title: java打包成jar|执行jar包中的main方法
date: 2017-2-13 14:25:00
tags: [Ubuntu]
categories: Ubuntu
description: " "
---

java打包成jar
=====
```
jar -cvf [jar包的名字] [需要打包的文件]
```


执行jar包中的main方法
===
```
java -jar ****.jar
```
执行后总是运行指定的主方法，如果 jar 中有多个 main 方法，那么如何运行指定的 main 方法呢？
用下面的命令试试看：
```
java -classpath ****.jar ****.****.className [args]
“****.****”表示“包名”；
“className”表示“类名”；
“[args]”表示传入的参数；
```


直接运行 MANIFEST.MF 中指定的 main 方法：
```
java -jar mplus-service-jar-with-dependencies.jar
```
运行指定的 main 方法（MANIFEST.MF 中没有指定的main方法）：
```
java -cp mplus-service-jar-with-dependencies.jar com.smbea.dubbo.bin.Console start
```

想阅读作者的更多文章，可以查看我的公共号：
<center>![振兴书城](http://upload-images.jianshu.io/upload_images/1319879-612c4c66d40ce855.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</center>