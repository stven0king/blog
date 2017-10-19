---
title: Retrofit2.0中注解使用套路
date: 2016-07-11 10:08:43
tags: [Retrofit2.0,Android网络请求框架]
categories: Android
description: "之前有讲过Retrofit2.0的简单使用和解析。最近在做Retrofit替换之前使用的AsyncHttpClient，在替换的过程中遇到一些之前忽视的小细节。自己感觉知道这几点在开发中灵活使用Retrofit非常有好处。"
---

之前有讲过Retrofit2.0的简单使用和解析。最近在做Retrofit替换之前使用的AsyncHttpClient，在替换的过程中遇到一些之前忽视的小细节。自己感觉知道这几点在开发中灵活使用Retrofit非常有好处。
#说说Retrofit中的注解
> @Query，@QueryMap，@Field，@FieldMap，@FormUrlEncoded，@Path，@Url
这七种注解应该是最常用的了。

下边列举各种应用场景。
一、get方式请求静态url地址
===
```
Retrofit retrofit = new Retrofit.Builder()
	.baseUrl("https://api.github.com/")
    .build();
public interface GitHubService {
	//无参数
	@GET("users/stven0king/repos")
	Call<List<Repo>> listRepos();
	//少数参数
	@GET("users/stven0king/repos")
	Call<List<Repo>> listRepos(@Query("time") long time);
	//参数较多
	@GET("users/stven0king/repos")
	Call<List<Repo>> listRepos(@QueryMap Map<String, String> params);
}
```
@Query和@QueryMap也可以结合在一起使用。
> 要是对应的url在服务端支持get/post两种类型的请求的话，那么上面的@GET变为@POST也可以执行，只不过post请求时所带的参数也会像get方式一样已？key=value&key1=vaule2...的形式拼接在url的后边。

二、post方式请求静态url地址
===
```
Retrofit retrofit = new Retrofit.Builder()
    .baseUrl("https://api.github.com/")
    .build()
public interface GitHubService {
	//无参数
	@POST("users/stven0king/repos")
	Call<List<Repo>> listRepos();
	//少数参数
	@FormUrlEncoded
	@POST("users/stven0king/repos")
	Call<List<Repo>> listRepos(@Field("time") long time);
	//参数较多
	@FormUrlEncoded
	@POST("users/stven0king/repos")
	Call<List<Repo>> listRepos(@FieldMap Map<String, String> params);
}
```
@Field和@FieldMap可以结合在一起使用。
> 另外是不是发现了比起@GET多了一个@FromUrlEncoded的注解。如果去掉@FromUrlEncoded在post请求中使用@Field和@FieldMap，那么程序会抛出 [java.lang.IllegalArgumentException: @Field parameters can only be used with form encoding. (parameter #1)]的错误异常。如果将@FromUrlEncoded添加在@GET上面呢，同样的也会抛出 [java.lang.IllegalArgumentException:FormUrlEncoded can only be specified on HTTP methods with request body (e.g., @POST).] 的错误异常。


三、半静态的url地址请求
===
```java
Retrofit retrofit = new Retrofit.Builder()
    .baseUrl("https://api.github.com/")
    .build()
	
public interface GitHubService {
  @GET("users/{user}/repos")
  Call<List<Repo>> listRepos(@Path("user") String user);
}
```

四、动态的url地址请求
===
```java
Retrofit retrofit = new Retrofit.Builder()
    .baseUrl("https://api.github.com/")
    .build()
	
public interface GitHubService {
  @GET
  Call<List<Repo>> listRepos(@Url String user);
}
```
五、总结小细节
===
* 当@GET或@POST注解的url为全路径时（可能和baseUrl不是一个域），会直接使用注解的url的域。
*  如果请求为post实现，那么最好传递参数时使用@Field、@FieldMap和@FormUrlEncoded。因为@Query和或QueryMap都是将参数拼接在url后面的，而@Field或@FieldMap传递的参数时放在请求体的。
* 使用@Path时，path对应的路径不能包含"/"，否则会将其转化为%2F。在遇到想动态的拼接多节url时，还是使用@Url吧。

想阅读作者的更多文章，可以查看我的公共号：
<center>![振兴书城](http://upload-images.jianshu.io/upload_images/1319879-612c4c66d40ce855.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</center>