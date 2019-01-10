---
title: Android平台JSON预览(JSON-handle)
date: 2018-09-10 20:38:43
tags: [JSON]
categories: Android
description:  "最近在做接口加密，所有的数据（`request`和`response`）都是加密数据，无法沟通`fildder`或者`Charles`抓包查看。那么自己做一个查看`json``格式的View`:支持动态的放大，缩小，支持所有数据格式~！！"
---

# [个人博客地址 http://dandanlove.com/](http://dandanlove.com/)

# JSON-handle

Chrome常用的插件[JSON-handle](https://chrome.google.com/webstore/detail/json-handle/iahnhfdhidomcpggpaimmmahffihkfnj)，用过的都知道。
最近在做接口加密，所有的数据（`request`和`response`）都是加密数据，无法沟通`fildder`或者`Charles`抓包查看。那么自己做一个查看`json``格式的View`:支持动态的放大，缩小，支持所有数据格式~！

<center>![json-handle.png](https://upload-images.jianshu.io/upload_images/1319879-74b659c715cf5b13.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/620)</center>

效果图：

<center>![json-handle.jpg](https://upload-images.jianshu.io/upload_images/1319879-4abc3f14d2b77efe.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</center>

GitHub地址： [JsonHandleView](https://github.com/stven0king/JsonHandleView)

## 依赖

```
implementation 'com.tzx.json:jsonhandleview:1.0.0'
```

## 使用

```xml
<?xml version="1.0" encoding="utf-8"?>
<FrameLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:fillViewport="true"
    android:orientation="vertical">

    <com.dandan.jsonhandleview.library.JsonViewLayout
        android:id="@+id/jsonView"
        android:layout_width="match_parent"
        android:layout_height="wrap_content" />

</FrameLayout>
```

```java
JsonViewLayout jsonViewLayout = findViewById(R.id.jsonView);
jsonViewLayout.bindJson("your json strings." || JSONObject || JSONArray);
```

## 自定义风格

```java
// Color
jsonViewLayout.setKeyColor()
jsonViewLayout.setObjectKeyColor()
jsonViewLayout.setValueTextColor()
jsonViewLayout.setValueNumberColor()
jsonViewLayout.setValueNullColor()
jsonViewLayout.setValueBooleanColor()
jsonViewLayout.setArrayLengthColor()

// TextSize
jsonViewLayout.setTextSize()
```

文章到这里就全部讲述完啦，若有其他需要交流的可以留言哦~！~！

想阅读作者的更多文章，可以查看我 [个人博客](http://dandanlove.com/) 和公共号：

<center>![振兴书城](http://upload-images.jianshu.io/upload_images/1319879-612c4c66d40ce855.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</center>