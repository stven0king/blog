---
title: Android项目解耦--路由框架ARouter的使用
date: 2018-02-06 10:58:00
tags: [ARouter]
categories: Android
description: "随着业务量的增长，客户端必然随之越来越业务和功能模块耦合越来越生，开发人员代码维护成本越来越高。App一般都会走向组件化、插件化的道路，而组件化、插件化的前提就是解耦。"
---

[Android项目解耦--路由框架ARouter源码解析](http://dandanlove.com/2018/02/06/arouter-source/)
## 前言
随着业务量的增长，客户端必然随之越来越业务和功能模块耦合越来越生，开发人员代码维护成本越来越高。
App一般都会走向组件化、插件化的道路，而组件化、插件化的前提就是解耦，那么我们首先要做的就是解耦页面之间的依赖关系。

<center>![2.9.png](http://upload-images.jianshu.io/upload_images/1319879-1cd8adf59a67340d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/620)</center>

## 目前Android原生页面跳转现状

- （显式的startActivity）多个module之前的页面跳转必须使module之间进行依赖；
- （隐式的intent-filter）书写麻烦，不好管理成功与否难以控制；
- Native页&M页跳转统一Native页面有不同的协议，管理复杂；
- 页面跳转过程无法干预（增加拦截过滤，日志打点）；
- 页面跳转结果无法修改（跳转失败进行重定向）；

## 模块解耦&高效开发

- "声明/使用" 简单.
- 适用多module开发,避免直接依赖.
- 统一协议, 适用"H5/Weex/Native" 跳转 "Native",对"Android/ios"两个平台协议应该是一样的.
- 有统一的外部调用入口
- 能对"不支持"的跳转统一处理
- 支持跳转前预处理
- 支持重定向


## ARouter现在有的模块解耦的最好的轮子

[ARouter](https://github.com/alibaba/ARouter) git上star四千多。

### ARouter应用场景

>- 从外部URL映射到内部页面，以及参数传递与解析
>- 跨模块页面跳转，模块间解耦
>- 拦截跳转过程，处理登陆、埋点等逻辑
>- 跨模块API调用，通过控制反转来做组件解耦

### ARouter的已支持功能

- 支持直接解析标准URL进行跳转，并自动注入参数到目标页面中
- 支持多模块工程使用
- 支持添加多个拦截器，自定义拦截顺序
- 支持依赖注入，可单独作为依赖注入框架使用
- 支持InstantRun(本人使用时貌似有问题无法找到该类`com.android.tools.fd.runtime.Paths`)
- 支持MultiDex(Google方案)
- 映射关系按组分类、多级管理，按需初始化
- 支持用户指定全局降级与局部降级策略
- 页面、拦截器、服务等组件均自动注册到框架
- 支持多种方式配置转场动画
- 支持获取Fragment

### ARouter项目集成

#### 添加依赖配置

```java
android {
    defaultConfig {
	...
	javaCompileOptions {
	    annotationProcessorOptions {
		arguments = [ moduleName : project.getName() ]
	    }
	}
    }
}

dependencies {
    // 替换成最新版本, 需要注意的是api
    // 要与compiler匹配使用，均使用最新版可以保证兼容
    compile 'com.alibaba:arouter-api:x.x.x'
    annotationProcessor 'com.alibaba:arouter-compiler:x.x.x'
    ...
}
// 旧版本gradle插件(< 2.2)，可以使用apt插件，配置方法见文末'其他#4'
// Kotlin配置参考文末'其他#5'
```

#### 详细的API说明
```java
// 构建标准的路由请求
ARouter.getInstance().build("/home/main").navigation();

// 构建标准的路由请求，并指定分组
ARouter.getInstance().build("/home/main", "ap").navigation();

// 构建标准的路由请求，通过Uri直接解析
Uri uri;
ARouter.getInstance().build(uri).navigation();

// 构建标准的路由请求，startActivityForResult
// navigation的第一个参数必须是Activity，第二个参数则是RequestCode
ARouter.getInstance().build("/home/main", "ap").navigation(this, 5);

// 直接传递Bundle
Bundle params = new Bundle();
ARouter.getInstance()
	.build("/home/main")
	.with(params)
	.navigation();

// 指定Flag
ARouter.getInstance()
	.build("/home/main")
	.withFlags();
	.navigation();

// 获取Fragment
Fragment fragment = (Fragment) ARouter.getInstance().build("/test/fragment").navigation();
					
// 对象传递
ARouter.getInstance()
	.withObject("key", new TestObj("Jack", "Rose"))
	.navigation();

// 觉得接口不够多，可以直接拿出Bundle赋值
ARouter.getInstance()
	    .build("/home/main")
	    .getExtra();

// 转场动画(常规方式)
ARouter.getInstance()
    .build("/test/activity2")
    .withTransition(R.anim.slide_in_bottom, R.anim.slide_out_bottom)
    .navigation(this);

// 转场动画(API16+)
ActivityOptionsCompat compat = ActivityOptionsCompat.
    makeScaleUpAnimation(v, v.getWidth() / 2, v.getHeight() / 2, 0, 0);

// ps. makeSceneTransitionAnimation 使用共享元素的时候，需要在navigation方法中传入当前Activity

ARouter.getInstance()
	.build("/test/activity2")
	.withOptionsCompat(compat)
	.navigation();
        
// 使用绿色通道(跳过所有的拦截器)
ARouter.getInstance().build("/home/main").greenChannel().navigation();

// 使用自己的日志工具打印日志
ARouter.setLogger();
```

#### 添加注解

```java
// 在支持路由的页面上添加注解(必选)
// 这里的路径需要注意的是至少需要有两级，/xx/xx
@Route(path = "/test/activity")
public class YourActivity extend Activity {
    ...
}
```

#### 初始化SDK

```java
if (isDebug()) {           // 这两行必须写在init之前，否则这些配置在init过程中将无效
    ARouter.openLog();     // 打印日志
    ARouter.openDebug();   // 开启调试模式(如果在InstantRun模式下运行，必须开启调试模式！线上版本需要关闭,否则有安全风险)
}
ARouter.init(mApplication); // 尽可能早，推荐在Application中初始化
```

#### 发起路由操作

```java
// 1. 应用内简单的跳转(通过URL跳转在'进阶用法'中)
ARouter.getInstance().build("/test/activity").navigation();

// 2. 跳转并携带参数
ARouter.getInstance().build("/test/1")
			.withLong("key1", 666L)
			.withString("key3", "888")
			.withObject("key4", new Test("Jack", "Rose"))
			.navigation();
```

#### 添加混淆规则(如果使用了Proguard)

```java
-keep public class com.alibaba.android.arouter.routes.**{*;}
-keep class * implements com.alibaba.android.arouter.facade.template.ISyringe{*;}

# 如果使用了 byType 的方式获取 Service，需添加下面规则，保护接口
-keep interface * implements com.alibaba.android.arouter.facade.template.IProvider

# 如果使用了 单类注入，即不定义接口实现 IProvider，需添加下面规则，保护实现
-keep class * implements com.alibaba.android.arouter.facade.template.IProvider
```

#### 通过URL跳转
html内容
```html
<html>
<head lang="en">
    <meta charset="UTF-8">
    <title>测试</title>
    <script type="text/javascript">
        function openActivity() {
            window.location.href = 'router://dandanlove/activity/ParamsCallActivity?obj={"age":19,"name":"Jack"}';
        }
    </script>
</head>
<body>
<div class="testDiv">
    <button onclick="openActivity()">openActivity</button><br>
    <a href='router://dandanlove/activity/ParamsCallActivity?name=Tom&age=18&girl=true&obj={"age":19,"name":"Jack"}'>ExternalCallActivity</a>
</div>
</body>
</html>
```
AndroidManifest.xml注册拦截
```java
<activity android:name="com.dandan.tzx.activity.SchameFilterActivity">
    <intent-filter>
        <data
            android:host="dandanlove"
            android:scheme="router"/>
        <action android:name="android.intent.action.VIEW"/>
        <category android:name="android.intent.category.DEFAULT"/>
        <category android:name="android.intent.category.BROWSABLE"/>
    </intent-filter>
</activity>
```
中转页面和目标注解页面
```java
// 新建一个Activity用于监听Schame事件,之后直接把url传递给ARouter即可
public class SchameFilterActivity extends Activity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
	super.onCreate(savedInstanceState);
	Uri uri = getIntent().getData();
	ARouter.getInstance().build(uri).navigation();
	finish();
    }
}
```
#### 解析URL中的参数
```java
@Router(path = "/activity/ParamsCallActivity")
public class ParamsCallActivity extends Activity {
    @Autowired
    public String name;
    @Autowired
    int age;
    @Autowired(name = "girl") // 通过name来映射URL中的不同参数
    boolean boy;
    @Autowired
    TestObj obj;    // 支持解析自定义对象，URL中使用json传递

    @Override
    protected void onCreate(Bundle savedInstanceState) {
	super.onCreate(savedInstanceState);
	ARouter.getInstance().inject(this);

	// ARouter会自动对字段进行赋值，无需主动获取
	Log.d("param", name + age + boy);
    }
}
```
#### 自定义序列化服务
```java
// 如果需要传递自定义对象，需要实现 SerializationService,并使用@Route注解标注(方便用户自行选择序列化方式)，例如：
@Route(path = "/service/json")
public class JsonServiceImpl implements SerializationService {
    @Override
    public void init(Context context) {

    }

    @Override
    public <T> T json2Object(String text, Class<T> clazz) {
        return JSON.parseObject(text, clazz);
    }

    @Override
    public String object2Json(Object instance) {
        return JSON.toJSONString(instance);
    }
}
```
#### 声明拦截器(拦截跳转过程，面向切面编程)
```java
// 比较经典的应用就是在跳转过程中处理登陆事件，这样就不需要在目标页重复做登陆检查
// 拦截器会在跳转之间执行，多个拦截器会按优先级顺序依次执行
@Interceptor(priority = 8, name = "测试用拦截器")
public class TestInterceptor implements IInterceptor {
    @Override
    public void process(Postcard postcard, InterceptorCallback callback) {
	...
	callback.onContinue(postcard);  // 处理完成，交还控制权
	// callback.onInterrupt(new RuntimeException("我觉得有点异常"));      // 觉得有问题，中断路由流程

	// 以上两种至少需要调用其中一种，否则不会继续路由
    }

    @Override
    public void init(Context context) {
	// 拦截器的初始化，会在sdk初始化的时候调用该方法，仅会调用一次
    }
}
```
#### 处理跳转结果
```java
// 使用两个参数的navigation方法，可以获取单次跳转的结果
ARouter.getInstance().build("/test/1").navigation(this, new NavigationCallback() {
    @Override
    public void onFound(Postcard postcard) {
      ...
    }

    @Override
    public void onLost(Postcard postcard) {
	...
    }
});
```
#### 自定义全局降级策略
```java
// 实现DegradeService接口，并加上一个Path内容任意的注解即可
@Route(path = "/xxx/xxx")
public class DegradeServiceImpl implements DegradeService {
  @Override
  public void onLost(Context context, Postcard postcard) {
	// do something.
  }

  @Override
  public void init(Context context) {

  }
}
```
#### 为目标页面声明更多信息
```java
// 我们经常需要在目标页面中配置一些属性，比方说"是否需要登陆"之类的
// 可以通过 Route 注解中的 extras 属性进行扩展，这个属性是一个 int值，换句话说，单个int有4字节，也就是32位，可以配置32个开关
// 剩下的可以自行发挥，通过字节操作可以标识32个开关，通过开关标记目标页面的一些属性，在拦截器中可以拿到这个标记进行业务逻辑判断
@Route(path = "/test/activity", extras = Consts.XXXX)
```

#### 通过依赖注入解耦:服务管理(一) 暴露服务
```java
// 声明接口,其他组件通过接口来调用服务
public interface HelloService extends IProvider {
    String sayHello(String name);
}

// 实现接口
@Route(path = "/service/hello", name = "测试服务")
public class HelloServiceImpl implements HelloService {

    @Override
    public String sayHello(String name) {
	return "hello, " + name;
    }

    @Override
    public void init(Context context) {

    }
}
```

#### 通过依赖注入解耦:服务管理(二) 发现服务
```java
public class Test {
    @Autowired
    HelloService helloService;

    @Autowired(name = "/service/hello")
    HelloService helloService2;

    HelloService helloService3;

    HelloService helloService4;

    public Test() {
	ARouter.getInstance().inject(this);
    }

    public void testService() {
	 // 1. (推荐)使用依赖注入的方式发现服务,通过注解标注字段,即可使用，无需主动获取
	 // Autowired注解中标注name之后，将会使用byName的方式注入对应的字段，不设置name属性，会默认使用byType的方式发现服务(当同一接口有多个实现的时候，必须使用byName的方式发现服务)
	helloService.sayHello("Vergil");
	helloService2.sayHello("Vergil");

	// 2. 使用依赖查找的方式发现服务，主动去发现服务并使用，下面两种方式分别是byName和byType
	helloService3 = ARouter.getInstance().navigation(HelloService.class);
	helloService4 = (HelloService) ARouter.getInstance().build("/service/hello").navigation();
	helloService3.sayHello("Vergil");
	helloService4.sayHello("Vergil");
    }
}
```

#### 重写跳转URL
```java
// 实现PathReplaceService接口，并加上一个Path内容任意的注解即可
@Route(path = "/xxx/xxx") // 必须标明注解
public class PathReplaceServiceImpl implements PathReplaceService {
    /**
     * For normal path.
     *
     * @param path raw path
     */
    String forString(String path) {
	return path;    // 按照一定的规则处理之后返回处理后的结果
    }

   /**
    * For uri type.
    *
    * @param uri raw uri
    */
   Uri forUri(Uri uri) {
	return url;    // 按照一定的规则处理之后返回处理后的结果
   }
}
```

文章到这里就全部讲述完啦，若有其他需要交流的可以留言哦~！~！

想阅读作者的更多文章，可以查看我 [个人博客](http://dandanlove.com/) 和公共号：
<center>![振兴书城](http://upload-images.jianshu.io/upload_images/1319879-612c4c66d40ce855.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</center>