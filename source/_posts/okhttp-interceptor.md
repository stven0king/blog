---
title: Okhttp拦截器Interceptor学习和使用
date: 2018-11-25 18:24:43
tags: [interceptor,okhttp]
categories: Android
description:  "前年的这个时候我们项目将网络框架替换为`okhttp+retrofit` ，然后我对 `retrofit` 源码进行了学习和分享，写了几篇相关的文章同时更新了项目的网络框架。需求是推动任何事物向前发展的动力，这次我们项目需要对网络接口进行加密了，开发过程涉及到了okhttp的网路层的处理，所以我又将其源码翻了一番。"
---


# 前言

前年的这个时候我们项目将网络框架替换为`okhttp+retrofit` ，然后我对 `retrofit` 源码进行了学习和分享，写了几篇相关的文章同时更新了项目的网络框架。

 [Android网络之Retrofit2.0使用和解析](https://blog.csdn.net/stven_king/article/details/51839537)
 [Retrofit2.0中注解使用套路](https://blog.csdn.net/stven_king/article/details/52372172)
 [Retrofit2.0+Okhttp不依赖服务端的数据缓存](https://blog.csdn.net/stven_king/article/details/52577248)

需求是推动任何事物向前发展的动力，这次我们项目需要对网络接口进行加密了，开发过程涉及到了okhttp的网路层的处理，所以我又将其源码翻了一番。

回顾一下我们曾经学习过的因特网五层协议栈：
>- 网络请求发出时：应用层->传输层->网络层->连接层->物理层 
>- 收到响应后：物理层->连接层->网络层->传输层->应用层

这个很像我们这次要讲的 `okhttp` 中的`interceptor` 的责任链模式。

# Interceptor

[Interceptor的wiki](https://github.com/square/okhttp/wiki/Interceptors)

按照以往的惯例我们先上图，然后在对每个步骤进行详细的讲述。

![okhttp-interceptors](http://upload-images.jianshu.io/upload_images/1319879-e9fb6406fad1972c?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## 为什么会有拦截器

我们在进行应用开发的时候都会在请求中增加一些我们应用需要和服务端交互的通用信息，比如在 `header` 中增加用户的登录态信息等等。或者像  [Retrofit2.0+Okhttp不依赖服务端的数据缓存](https://blog.csdn.net/stven_king/article/details/52577248) 这篇文章中不依赖服务端的缓存，在请求的过程中我们自己修改一些请求的 `request` 和 `response` 。

这个时候拦截器就是我们的强大的助力。

## okhttp中的拦截器

我们从 `okhttp` 处理一条普通的url请求的代码执行过程中观察 `interceptors` 的工作。

下边的代码是从 [okhttp官网](//http://square.github.io/okhttp/) 摘的一段示例代码：

```java
OkHttpClient client = new OkHttpClient();

String run(String url) throws IOException {
  Request request = new Request.Builder()
      .url(url)
      .build();

  Response response = client.newCall(request).execute();
  return response.body().string();
}
```

我们跟进 `okhttpclient.newcall` 方法：

```java
public class OkHttpClient implements Cloneable, Call.Factory, WebSocket.Factory {
    /***部分代码省略***/
	/**
     * Prepares the {@code request} to be executed at some point in the future.
     */
    @Override 
    public Call newCall(Request request) {
    	return new RealCall(this, request, false /* for web socket */);
  	}
}
```

获取一个真正用来进行请求的Call `RealCall.execute` :

```java
final class RealCall implements Call {
	final RetryAndFollowUpInterceptor retryAndFollowUpInterceptor;
	RealCall(OkHttpClient client, Request originalRequest, boolean forWebSocket) {
	    final EventListener.Factory eventListenerFactory = client.eventListenerFactory();
	    this.client = client;
	    this.originalRequest = originalRequest;
	    this.forWebSocket = forWebSocket;
	    this.retryAndFollowUpInterceptor = new RetryAndFollowUpInterceptor(client, forWebSocket);
	    // TODO(jwilson): this is unsafe publication and not threadsafe.
	    this.eventListener = eventListenerFactory.create(this);
	}
	@Override 
	public Response execute() throws IOException {
	    synchronized (this) {
	    	if (executed) throw new IllegalStateException("Already Executed");
	      	executed = true;
	    }
	    captureCallStackTrace();
	    try {
	        //进行网络请求
	      	client.dispatcher().executed(this);
	      	//经过一层层网络拦截器之后，获取网络请求的返回值
	      	Response result = getResponseWithInterceptorChain();
	      	if (result == null) throw new IOException("Canceled");
	      	return result;
	    } finally {
	      	client.dispatcher().finished(this);
	    }
	  }
	  Response getResponseWithInterceptorChain() throws IOException {
	    // Build a full stack of interceptors.
	    List<Interceptor> interceptors = new ArrayList<>();
	    //Application拦截器
	    interceptors.addAll(client.interceptors());
	    //重定向和失败后重新请求拦截器
	    interceptors.add(retryAndFollowUpInterceptor);
	    //网桥拦截器，顾名思义client和Server之前的桥梁
	    interceptors.add(new BridgeInterceptor(client.cookieJar()));
	    //缓存处理拦截器
	    interceptors.add(new CacheInterceptor(client.internalCache()));
	    //Socket层的握手链接
	    interceptors.add(new ConnectInterceptor(client));
	    if (!forWebSocket) {
	    	//网络烂拦截器
	    	interceptors.addAll(client.networkInterceptors());
	    }
	    //client和Server之前的读写操作
	    interceptors.add(new CallServerInterceptor(forWebSocket));
		//责任链开始执行
	    Interceptor.Chain chain = new RealInterceptorChain(
	        interceptors, null, null, null, 0, originalRequest);
	    return chain.proceed(originalRequest);
	  }
}
```

## Interceptor介绍

我们先看拦截器的接口定义：

```java
public interface Interceptor {
	//拦截处理
	Response intercept(Chain chain) throws IOException;
	interface Chain {
		//获取请求的request
	  	Request request();
		//处理request获取response
	    Response proceed(Request request) throws IOException;
	
	    /**
	     * Returns the connection the request will be executed on. This is only available in the chains
	     * of network interceptors; for application interceptors this is always null.
	     */
	    @Nullable Connection connection();
	}
}
```

在我们跟踪源码的执行的过程我们回忆下最开始的时候的流程图，涉及到的拦截器以及他们各自的位置和在网络请求的作用。

> Application Interceptor

我们可以自定义设置 `Okhttp` 的拦截器之一。

从流程图中我们可以看到一次网络请求它只会执行一次拦截，而且它是第一个触发拦截的，这里拦截到的url请求的信息都是最原始的信息。所以我们可以在该拦截器中添加一些我们请求中需要的通用信息，打印一些我们需要的日志。

当然我们可以定义多个这样的拦截器，一个处理 `header` 信息，一个处理 `接口请求的 加解密`  。

> NetwrokInterceptor

`NetwrokInterceptor` 也是我们可以自定义的拦截器之一。 

它位于倒数第二层，会经过 `RetryAndFollowIntercptor` 进行重定向并且也会通过 `BridgeInterceptor` 进行 `request `请求头和 响应 `resposne` 的处理，因此这里可以得到的是更多的信息。在打印结果可以看到它内部重定向操作和失败重试，这里会有比 `Application Interceptor` 更多的日志。

> RetryAndFollowInterceptor

`RetryAndFollowUpInterceptor` 的作用，看到该拦截器的名称就知道，它就是一个负责失败重连的拦截器。它是 `Okhttp` 内置的第一个拦截器，通过 `while (true)` 的死循环来进行对异常结果或者响应结果判断是否要进行重新请求。

> BridgeInterceptor

`BridgeInterceptor` 为用户构建的一个 Request 请求转化为能够进行网络访问的请求，同时将网络请求回来的响应 Response 转化为用户可用的 Response。比如，涉及的网络文件的类型和网页的编码，返回的数据的解压处理等等。

> CacheInterceptor

`CacheInterceptor` 根据 `OkHttpClient` 对象的配置以及缓存策略对请求值进行缓存。

> ConnectInterceptor

`ConnectInterceptor` 在 OKHTTP 底层是通过 SOCKET 的方式于服务端进行连接的，并且在连接建立之后会通过 OKIO 获取通向 server 端的输入流 Source 和输出流 Sink。

> CallServerInterceptor

`CallServerInterceptor` 在 `ConnectInterceptor` 拦截器的功能就是负责与服务器建立 Socket 连接，并且创建了一个 HttpStream 它包括通向服务器的输入流和输出流。而接下来的 `CallServerInterceptor` 拦截器的功能使用 HttpStream 与服务器进行数据的读写操作的。


有关每个拦截器的具体代码分析：https://www.jianshu.com/u/8173f323f5bb


## 拦截器中的骚操作

```java
HttpUrl httpurl = request.url();
//获取request中的原始url地址
String requestUrl = httpurl.url().toString();
/**
 * 获取url中的参数
 * @param httpUrl
 * @return
 */
private Map<String, String> getHttpUrlParams(HttpUrl httpUrl) {
    Set<String> paramsSet = httpUrl.queryParameterNames();
    Map<String, String> paramMap = new HashMap<>();
    if (paramsSet != null && paramsSet.size() > 0) {
        for (String s: paramsSet) {
            paramMap.put(s, httpUrl.queryParameter(s));
        }
    }
    return paramMap;
}
//一般POST请求参数都是放在RequestBody中,使用时需要判断RequestBody是否为子类FormBody的实例
RequestBody requestBody = request.body();
/**
 * 获取请求form中的参数
 * @param formBody
 * @return
 */
private Map<String, String> getHttpUrlParams(FormBody formBody) {
    Map<String, String> paramMap = new HashMap<>();
    if (formBody != null) {
        for (int i = 0; i < formBody.size(); i++) {
            paramMap.put(formBody.name(i), formBody.value(i));
        }
    }
    return paramMap;
}
//构造新的POST表单
FormBody.Builder formBuilder = new FormBody.Builder();
for (String key: map.keySet()) {
    formBuilder.add(key, map.get(key));
}
//构造新的HttpUrl，动态修改请求url地址
HttpUrl httpUrl = HttpUrl.parse(requestUrlTrue);
//构造新的request请求
request = request.newBuilder()
        .method(POST, formBuilder.build())
        .url(httpUrl)
        .build();
//获取相应体对应的请求体，请求和返回一一对应
Request request = response.request()
//获取请求的相应体
ResponseBody responseBody = response.body();
//获取返回值类型
MediaType mediaType = responseBody.contentType();
//获取相应体重的数据流，只能获取一次，在获取之后数据流会关闭，再次获取会有异常抛出
byte[] responseBytes = responseBytes = responseBody.bytes();
//利用修改后的返回值，构造新的相应体
response = response.newBuilder()
        .body(ResponseBody.create(mediaType, responseBytes))
        .build();
```

文章到这里就全部讲述完啦，若有其他需要交流的可以留言哦~！~！

想阅读作者的更多文章，可以查看我 [个人博客](http://dandanlove.com/) 和公共号：

![振兴书城](http://upload-images.jianshu.io/upload_images/1319879-612c4c66d40ce855.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

