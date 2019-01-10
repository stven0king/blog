---
title: Android6.0动态权限适配&XMPermissions
date: 2018-09-29 18:50:43
tags: [Permission]
categories: Android
description:  "从 Android 6.0（API 级别 23）开始，用户开始在应用运行时向其授予权限，而不是在应用安装时授予。我们应用程序如果适配到6.0，那么就必须了解一下~！！"
---

# [个人博客地址 http://dandanlove.com/](http://dandanlove.com/)

# Android6.0动态权限适配&XMPermissions

<center>![permission.jpg](https://upload-images.jianshu.io/upload_images/1319879-ef9fa001f5927064.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/420)</center>


# Android6.0动态权限

## 简介

从 Android 6.0（API 级别 23）开始，用户开始在应用运行时向其授予权限，而不是在应用安装时授予。此方法可以简化应用安装过程，因为用户在安装或更新应用时不需要授予权限。它还让用户可以对应用的功能进行更多控制；例如，用户可以选择为相机应用提供相机访问权限，而不提供设备位置的访问权限。用户可以随时进入应用的“Settings”屏幕调用权限。摘自[Android官网:在运行时请求权限](https://developer.android.com/training/permissions/requesting?hl=zh-cn)。

## targetSdkVerion

我们在开发的时候需要指定`minSdkVersion` 和  `targetSdkVerion`。
>- `minSdkVersion`为app最低适配的版本，低于该版本的手机无法安装；

>-  `targetSdkVerion`简单来说就代表着你的App能够适配的系统版本，意味着你的App在这个版本的手机上做了充分的 前向 兼容性处理和实际测试。其实我们写代码时都是经常干这么一件事，就是 if(Build.VERSION.SDK_INT >= 23) { ... } ，这就是兼容性处理最典型的一个例子。如果你的target设置得越高，其实调用系统提供的API时，所得到的处理也是不一样的，甚至有些新的API是只有新的系统才有的;

## Android6.0特殊权限[Special Permissions](https://developer.android.com/guide/topics/permissions/overview)

看权限名就知道特殊权限比危险权限更危险，特殊权限需要在manifest中申请并且通过发送Intent让用户在设置界面进行勾。
```java
//SYSTEM_ALERT_WINDOW
private static final int REQUEST_CODE = 1;
private void requestAlertWindowPermission(){
    Intent intent = new Intent(Settings.ACTION_MANAGE_OVERLAY_PERMISSION);
    intent.setData(Uri.parse("package:"+ getPackageName()));
    startActivityForResult(intent, REQUEST_CODE);
}
@Override
protected void onActivityResult(intrequestCode, intresultCode, Intent data){
    super.onActivityResult(requestCode, resultCode, data);
    if(requestCode == REQUEST_CODE) {
        if(Settings.canDrawOverlays(this)) {
            Log.i(LOGTAG, "onActivityResult granted");
        }
    }
}
//WRITE_SETTINGS 修改系统设置
private static final int REQUEST_CODE_WRITE_SETTINGS = 2;
private void requestWriteSettings(){
    Intent intent = new Intent(Settings.ACTION_MANAGE_WRITE_SETTINGS);
    intent.setData(Uri.parse("package:"+ getPackageName()));
    startActivityForResult(intent, REQUEST_CODE_WRITE_SETTINGS );
}
@Override
protected void onActivityResult(intrequestCode, intresultCode, Intent data){
    super.onActivityResult(requestCode, resultCode, data);
    if(requestCode == REQUEST_CODE_WRITE_SETTINGS) {
        if(Settings.System.canWrite(this)) {
            Log.i(LOGTAG, "onActivityResult write settings granted");
        }
    }
}
```

## Android6.0普通权限[normal permission](https://developer.android.com/guide/topics/permissions/overview)

普通权限不会对用户的隐私和安全产生太大的风险，所以只需要在AndroidManifest.xml中声明即可.

## Android6.0危险权限[dangerous permission](https://developer.android.com/guide/topics/permissions/overview)

- Normal Permission：写在xml文件里，那么App安装时就会默认获得这些权限，即使是在Android6.0系统的手机上，用户也无法在安装后动态取消这些normal权限，这和以前的权限系统是一样的，不变。
- Dangerous Permission：还是得写在xml文件里，但是App安装时具体如果执行授权分以下几种情况：
    - 1、targetSDKVersion < 23 & API(手机系统) < 6.0 ：安装时默认获得权限，且用户无法在安装App之后取消权限。
    - 3、targetSDKVersion < 23 & API(手机系统) >= 6.0 ：安装时默认获得权限，但是用户可以在安装App完成后动态取消授权（ 取消时手机会弹出提醒，告诉用户这个是为旧版手机打造的应用，让用户谨慎操作 ）。
    - 2、targetSDKVersion >= 23 & API(手机系统) < 6.0 ：安装时默认获得权限，且用户无法在安装App之后取消权限。
    - 4、targetSDKVersion >= 23 & API(手机系统) >= 6.0 ：安装时不会获得权限，可以在运行时向用户申请权限。用户授权以后仍然可以在设置界面中取消授权，用户主动在设置界面取消后，在app运行过程中可能会出现crash。

## Dangerous permissions and permission groups(危险权限和权限组)

> 同一组的任何一个权限被授权了，其他权限也自动被授权。例如，一旦WRITE_CONTENTS被授权了，APP也有READ_CONTACTS和GET_ACCOUNTS了。

|permission-group	|dangerous permissions|
|:--|:--|
|CALENDAR(日历)	|READ_CALENDAR , WRITE_CALENDAR|
|CAMERA(照相机)	|CAMERA|
|CONTACTS(联系人)|	READ_CONTACTS , WRITE_CONTACTS , GET_ACCOUNTS|
|LOCATION(位置)	|ACCESS_FINE_LOCATION (访问精细的位置)， ACCESS_COARSE_LOCATION(访问粗略的位置)|
|MICROPHONE(麦克风)	|RECORD_AUDIO(录音)|
|PHONE(手机)	|READ_PHONE_STATE ， CALL_PHONE ， READ_CALL_LOG ， WRITE_CALL_LOG ， ADD_VOICEMAIL(添加语音信箱) ， USE_SIP(使用SIP协议 ， PROCESS_OUTGOING_CALLS(程序拨出电话)|
|SENSORS(传感器)|	BODY_SENSORS|
|SMS|	SEND_SMS ， RECEIVE_SMS ， READ_SMS ， RECEIVE_WAP_PUSH ， RECEIVE_MMS|
|TORAGE(存储)|	READ_EXTERNAL_STORAGE ，WRITE_EXTERNAL_STORAGE|

## 如何开始动态申请权限

### 判断权限是否具有某项权限
```
ContextCompat.checkSelfPermission(Context context,String permission);
ActivityCompat.checkSelfPermission(Context context, String permission);
activity.checkSelfPermission(String permission);
```

### 申请权限
```
ActivityCompat.requestPermissions(Activity activity,String[] permissions,int requestCode);
activity.requestPermissions(String[] permissions, int requestCode);
//申请权限回调方法,在Activity或Fragment重写
onRequestPermissionsResult(int requestCode, String[] permissions, int[] grantResults)
```

### 是否要提示用户申请该权限的缘由，sdk小于23恒为false
```
ActivityCompat.shouldShowRequestPermissionRationale(Activity activity, String permission)
0、之前没有拒绝过此权限的申请（第一次安装后请求权限前调用）：false
1、曾经被拒绝过权限后再调用：true
2、曾经被拒绝过权限且不再询问后再调用：false
3、系统不允许任何程序获取该权限：false
4、查看源码得知安卓6.0以下返回：false
5、总是允许权限后再次调用：false
```

## 在APP使用过程中，从设置中更改权限

如果应用程序的某个业务逻辑需要使用权限，但用户没有选择开启。那么最好引导用户去设置界面修改应用程序的权限。

# XMPermissions

## 导读
如果我们应用需要动态申请危险权限，按照Google官方问档我们需要在`activity`或者`fragment`中的`onRequestPermissionsResult `方法进行回调处理。一个执行任务代码需要分开写在两处地方，这我们的代码会变得很不优雅。

有没有链式、流式或者注解的方式去解决这个问题？有而且很多，以下是我在`github` 上找的`start` 最多的开源库。

[RxPermissions](https://github.com/tbruyelle/RxPermissions.git) RxJava流式用法；
[XXPermissions](https://github.com/getActivity/XXPermissions.git) 链式用法；
[permissions4m](https://github.com/jokermonn/permissions4m.git) 注解用法；

个人对注解使用不太感冒，而且项目有用到Rxjava的地方。
综上所述，我在`RxPermissions` 和 `XXPermissions` 基础上开发了 [XMPermissions](https://github.com/stven0king/XMPermission)。

## 依赖

```java
implementation 'com.tzx.lib:xmpermission:1.0.0'
```

## 使用

```java
public interface OnPermission {
    /**
     * 本次申请的权限全部通过
     */
    void hasPermission();

    /**
     * 本次申请的权限没有全部通过
     * @param granteds
     */
    void noPermission(List<PermissionState> granteds);
}
```

### RxJava


```java
XMPermissions.with(this)
    .request(Permission.Group.STORAGE, Permission.Group.PHONE)
    .subscribe(new SimpleSubscriber<List<PermissionState>>() {
        @Override
        public void onNext(List<PermissionState> permissionStates) {
            for (PermissionState s: permissionStates) {
                Log.d("tanzhenxing:", s.toString());
            }
        }
    });
```

### 链式调用

```java
XMPermissions.with(this)
                //.constantRequest() //可设置被拒绝后继续申请，直到用户授权或者永久拒绝
                .permission(Permission.Group.STORAGE, Permission.Group.CALENDAR) 
                .request(new OnPermission() {

                    @Override
                    public void hasPermission() {
                        Toast.makeText(MainActivity.this, "获取权限成功", Toast.LENGTH_SHORT).show();
                    }

                    @Override
                    public void noPermission(List<PermissionState> granteds) {
                        for (PermissionState s: granteds) {
                            Log.d("tzx:", s.toString());
                        }
                    }

                });
```

### 跳转到应用权限设置页面

```java
XMPermissions.gotoPermissionSettings(content);
```


# 6.0动态权限适配总结


有了[XMPermissions](https://github.com/stven0king/XMPermission) 适配6.0动态权限就非常简单了。将`targetVersion`升级到`23`，然后每个使用`储存、定位、电话、相机、录音`等危险权限的地方做权限的`check`。

当然这么做非常麻烦像`储存、定位、电话`这三个权限我们几乎每次接口访问都需要获取，所以我们可以将一些权限申请在应用启动前置。

- 转转：`储存、定位、电话`前置
- 58同城： `存储、电话`前置
- 京东： `定位、电话`前置
- 手机淘宝： `电话`前置
- 手机百度： `存储`前置

> 在进行短信发送和打电话时，不需要权限也可以哦~！我自己测试了4个主流厂商的8款手机。

随着`Android`系统的不断更新，后续后问题会继续同步哒~！


文章到这里就全部讲述完啦，若有其他需要交流的可以留言哦~！~！

想阅读作者的更多文章，可以查看我 [个人博客](http://dandanlove.com/) 和公共号：

<center>![振兴书城](http://upload-images.jianshu.io/upload_images/1319879-612c4c66d40ce855.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</center>