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
### 