---
layout: posts
title: okHttp解析
date: 2018-06-04 10:31:58
tags: [源码解析,笔记,android]
categories: 源码解析
---
## 介绍
OkHttp 现在作为Android时下最流行的网络框架(Retrofit 也是以OkHttp为底层的).先将okhttp的学习整理如下,本文是以3.10版本进行分析.(因为3.9版本跟3.10版本差别不大,有些图片的引用还是3.9的)

感谢: [json_it的博客](https://blog.csdn.net/json_it/article/details/78404010)  [码迷](http://www.mamicode.com/info-detail-2161332.html)

<img src = "http://okskqdic8.bkt.clouddn.com/okhttp_1.jpg" width = 500/>

## okhttp网络请求流程
以官网demo为例进行分析

<!-- more -->

<font color=#aaaaaa >Get请求</font>

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

<font color=#aaaaaa >Post请求</font>

```java
public static final MediaType JSON
    = MediaType.parse("application/json; charset=utf-8");
OkHttpClient client = new OkHttpClient();
String post(String url, String json) throws IOException {
  RequestBody body = RequestBody.create(JSON, json);
  Request request = new Request.Builder()
      .url(url)
      .post(body)
      .build();
  Response response = client.newCall(request).execute();
  return response.body().string();
}
```
### 调用流程图如下

<img src = "http://okskqdic8.bkt.clouddn.com/okhttp_2_1.png" width = 500/>
## 类,方法具体分析
### OkHttpClient
要完成一次请求,我们必须先有OkHttpClient实例,使用者的所有操作都是通过它来完成的.OkHttpClient用于协调各个子模块工作,各个子模块直接相互独立.OkHttpClient因为有众多子模块,所以使用建造者设计模式进行装配.获取OkHttpClient实例有两种方式

#### 方式1:

```java
	OkHttpClient client=new OkHttpClient(); 
```
不是说使用的是建造者模式吗?为啥没有Builder?
因为OkHttpClient的构造函数默认使用了默认的builder()

```java
	public OkHttpClient() {
    		this(new Builder());
 	}

	........

	public Builder() {
      		//调度器
      		dispatcher = new Dispatcher();
      		//请求协议 http1.1 http2
      		protocols = DEFAULT_PROTOCOLS;
      		// 链接规范 普通请求 加密请求(TLS)
      		connectionSpecs = DEFAULT_CONNECTION_SPECS;
      		//应用程序的HTTP调用的数量、大小和持续时间
      		eventListenerFactory =EventListener.factory(EventListener.NONE);
      		//选择在连接到URL引用的网络资源时要使用的代理服务器(如果有的话)
      		proxySelector = ProxySelector.getDefault();
      		//cookie
      		cookieJar = CookieJar.NO_COOKIES;
      		//java 套接字工厂
      		socketFactory = SocketFactory.getDefault();
      		//用于HTTPS请求时与域名校验
      		hostnameVerifier = OkHostnameVerifier.INSTANCE;
      		//https证书校验
      		certificatePinner = CertificatePinner.DEFAULT;
      		proxyAuthenticator = Authenticator.NONE;
      		authenticator = Authenticator.NONE;
      		//连接池
      		connectionPool = new ConnectionPool();
      		dns = Dns.SYSTEM;
      		followSslRedirects = true;
      		followRedirects = true;
      		retryOnConnectionFailure = true;
      		connectTimeout = 10_000;
      		readTimeout = 10_000;
      		writeTimeout = 10_000;
      		pingInterval = 0;
    	}

	........	

```
#### 方式2:

```java
	OkHttpClient.Builder bulider=new OKHttpClient.Builder(); 
	//根据builder进行自定义
	OkHttpClient client=buider.xxx().xxx().build();
```

注: 因为OkHttpClient内部有连接池,线程池等,所以一般来说一个应用就一个OkHttpClient实例.

### Request
Request类比较简单 只包含了 url, method, headers, body等一次请求的基本信息
### RealCall

### Dispatcher
### RetryAndFollowUpInterceptor
### BridgeInterceptor
### CacheInterceptor
### ConnectInterceptor
#### StreamAllocation
#### RealConnection
#### HttpCodec
### CallServerInterceptor
### RealInterceptorChain
### Response