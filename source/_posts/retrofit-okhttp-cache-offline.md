---
title: Retrofit2.0+Okhttp不依赖服务端的数据缓存
date: 2016-09-18 19:32:00
tags: [Retrofit2.0,Android网络请求框架]
categories: Android
description: "随着Retrofit在项目中的使用，替换的以前使用的网络框架，相关的缓存机制也要进行替换，网络上大部分的Retrofit+okhttp缓存资料都是进行针对所有url的而且需要服务端的配合。有些时候是先有服务然后app去调用这些服务，所以这个时候没有服务端的配合我们在前端实现缓存比较空难但并不是很可以。"
---
随着Retrofit在项目中的使用，替换的以前使用的网络框架，相关的缓存机制也要进行替换，网络上大部分的Retrofit+okhttp缓存资料都是进行针对所有url的而且需要服务端的配合。有些时候是先有服务然后app去调用这些服务，所以这个时候没有服务端的配合我们在前端实现缓存比较空难但并不是很可以。（举个列子，有一个原来的服务更本不支持cache，但是我们在app中需要缓存这个服务的数据，这应该是前一段时间替换网络库时最后遇到的问题）。

为什么要做缓存处理？
=====

客观回答：
> 减少服务器负荷，降低延迟提升用户体验。复杂的缓存策略会根据用户当前的网络情况采取不同的缓存策略，比如在2g网络很差的情况下，提高缓存使用的时间；不用的应用、业务需求、接口所需要的缓存策略也会不一样，有的要保证数据的实时性，所以不能有缓存，有的你可以缓存5分钟，等等。

自己回答：
> 在绝大部分的项目中我们前端开发人员只是考虑用户的流量，用户在产品性能上的体验。所以有时候服务端和前端没有依赖，即服务不支持缓存那么前端又需要缓存那么我们应该怎么做？普通的缓存模式已经很难适应这种需求了，下面将的就是利用Retrofit2.0+OkHttp3.0的缓存原理去实现我们的需求。

Retrofit+OkHttp的缓存机制：
=====

- 在 data/data/<包名>/cache 下建立一个用来进行数据存储的文件夹，保持缓存数据。
- 这样我们就可以在请求的时候，根据业务逻辑，请求网络数据或者读取缓存的数据。

缓存使用情况：
===

- 一般情况下无网络，数据从缓存中读取；
- 有网络则根据请求头，判断是请求网络还是读取缓存。

说到缓存，不是很了解的Http缓存的同学亦可以看一下[浏览器 HTTP 协议缓存机制详解](http://blog.csdn.net/stven_king/article/details/51899865) 这篇文章讲的很详细。

Cache控制：
=====
该部分对理解怎么缓存很重要~！

1、用在request中的cache控制头
>Pragma: no-cache :兼容早起HTTP协议版本 如1.0+
>Cache-Control: no-cache ，表示不希望得到一个缓存内容。只是希望，cache设备可能忽略。
>Cache-Control: no-store，表示client与server之间的设备不能缓存响应内容，并应该删除已有缓存。
>Cache-Control: only-if-cached，表示只接受是被缓存的内容

2、用在response中控制cache的头
>Cache-Control: max-age=3600，用相对于接收到的时间开始可缓存多久
>Cache-Control: s-maxage=3600，与上面类似，只是s-maxage一般用在cache服务器上，并只对public缓存有效
>Expires: Fri, 05 Jul 2002, 05:00:00 GMT基于GMT的时间，绝对时间，但该头容易受到本地错误时间影响
>Cache-Control: must-revalidate 该头表示内容可以被缓存但每次必须询问是否有更新。

[HTTP-请求、响应、缓存](https://cnbin.github.io/blog/2016/02/20/http-qing-qiu-,-xiang-ying-,-huan-cun/)


代码实现：
===

看到这里应该对缓存有一定的了解了，那么现在来看看怎么利用Retrofit2.0+Okhttp缓存的实现。

创建缓存文件，并对okhttp进行设置
```java
public class RetrofitApiFactory{
	private static OkHttpClient okHttpClient = null ;
	private static File cacheFile = new File(ImageUtils.getAppCacheDir(), "xxxCache");
	private static Cache cache = new Cache(cacheFile, 1024 * 1024 * 50);
    /******中间代码省略*****/
	public static void initOkHttpClient(){
		okHttpClient =
				new OkHttpClient.Builder()
						.cache(cache)
						.readTimeout(TIME_OUT, TimeUnit.SECONDS)
						.writeTimeout(TIME_OUT, TimeUnit.SECONDS)
						.addInterceptor(new JobInterceptor("Interceptor"))  
						//only 有网情况下，一分钟内每次请求都会重新请求，不会走缓存
						.addNetworkInterceptor(new JobInterceptor("NetworkInterceptor"))    
						//only 如果超过1分钟，离线请求不成功
						.build();
		clearCacheMap() ;
	}
}
```
控制Cache中最最最主要的部分：
```java
public class JobInterceptor implements Interceptor {
	/******中间代码省略*****/
    @Override
    public Response intercept(Chain chain) throws IOException {
        Request request = chain.request() ;
        if (!TextUtils.isEmpty(getCookie())) {
            if (!NetworkDetection.getIsConnected()) {
                try {
                    request = request
                            .newBuilder()
                            .addHeader("Cookie", getCookie())
                            .cacheControl(CacheControl.FORCE_CACHE)
                            .build();
                } catch (Exception e) {
                    e.printStackTrace();
                }
            } else {
                try {
                    request = request
                            .newBuilder()
                            .addHeader("Cookie", getCookie())
                            .build();
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        }
        Response response = null;
        try {
            response = chain.proceed(request);
            if(!response.isSuccessful()){
                RetrofitApiFactory.initOkHttpClient();
            } else {//如果请求体有缓存数据的需要那么对响应体进行缓存
                int maxAge = request.cacheControl().maxAgeSeconds();
                if (request.cacheControl().isPublic() && maxAge > 1) {
                    response = response.newBuilder()
                            .removeHeader("Pragma")//清楚响应体对Cache有影响的信息
                            .removeHeader("Cache-Control")//清楚响应体对Cache有影响的信息
                            .header("Cache-Control", "public, max-age=" + maxAge)
                            .build();
                }
            }
        } catch (IOException e) {
            RetrofitApiFactory.initOkHttpClient();
            throw new IOException(e) ;
        }

        return response;
    }
}
```
接口使用：
```java
@Headers("Cache-Control: public, max-age=86400")
@GET
Call<ResponseBody> getCacheData(@Url String url);
```

解析：
=====
上面的绝大部分内容大家都在类似的文章上看到的。这里主要讲一下几点：
> 一、我们所用的接口服务不支持缓存，所以我不能只修改头信息而让服务端返回的response响应体去实现数据本地缓存。当然在没有网络的情况下我们可以尝试去读取缓存。

>二、因为服务端没有提供response响应体的缓存，所以我们清除response响应体的Pragma、Cache-Control信息，然后根据自己设定的request请求体中的Cache信息去修改response响应体的Cache信息从而达到数据可以缓存。

>三、在开发的过程中遇到如果一个接口在某次请求返回404，那么以后的结果总是请求失败的404页面。所以在请求失败的时候需要初始化OkHttpClient实例。

如有问题，欢迎沟通指正~！~！
参考文章[Retrofit-Cache](http://blog.csdn.net/qqyanjiang/article/details/51316116)

想阅读作者的更多文章，可以查看我的公共号：
<center>![振兴书城](http://upload-images.jianshu.io/upload_images/1319879-612c4c66d40ce855.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</center>