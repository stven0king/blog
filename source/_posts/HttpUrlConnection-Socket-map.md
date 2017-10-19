---
title: Android网络之HttpUrlConnection和Socket关系图解
date: 2016-07-14 23:05:44
tags: [HttpUrlConnection,Socket,Android网络]
categories: Android
description: 多年以前Android的网络请求只有Apache开源的HttpClient和JDK的HttpUrlConnection，近几年随着OkHttp的流行Android在高版本的SDK中加入了OkHttp。但在Android官方文档中推荐使用HttpUrlConnection并且其会一直被维护，所以在学习Android网络相关的知识时我们队HttpUrlConnection要有足够的了解。。。。
---
最近在学习有关于网络方面的知识，研究Android网络请求框架。前几天阅读完Retrofit2.0源码写了一篇[Retrofit2.0使用和解析](http://blog.csdn.net/stven_king/article/details/51839537) 的文章，因为Retrofit2.0现在只支持OkHttp，OkHttp网络框架也在Android高版本的SDK中使用，自己为了能更好的优化Android中关于网络这个模块，然后又阅读了OkHttp3.0的源代码。OkHttp3.0的源码解析网上找的资料不很多，其中所用到的设计模式和网络相关的东西非常多。自己看的很懵逼，于是想先看看HttpUrlConnection的实现原理。

已经快23点了自己还在公司
先直接上图，明天再写源码分析吧
![HttpUrlConnection和Socket关系图解](http://img.blog.csdn.net/20160714225325530)

想阅读作者的更多文章，可以查看我的公共号：
<center>![振兴书城](http://upload-images.jianshu.io/upload_images/1319879-612c4c66d40ce855.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</center>