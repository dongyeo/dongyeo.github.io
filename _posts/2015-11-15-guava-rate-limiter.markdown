---
layout:     post
title:      "Guava RateLimiter 接口限流"
date:       2015-11-15 20:59:00
author:     "DongYeo"
header-img: "img/post-bg-04.jpg"
tags: ["Java","Guava"]
---

## 前言
开发过程中，难免会遇到需要对其他系统提供数据接口的情景。由于前期，接口的请求量不大，或者开发之前根本没有预期接口的请求量会到把系统搞挂的量级。为防止请求量过大，导致应用崩溃的情况出现，最好的措施就是对接口进行限流或者引流操作。

## 限流算法
常见的限流算法主要有两种：令牌桶算法和漏桶算法。
- 漏桶算法：如图packet先进入到漏桶里，漏桶以一定的速度出水（packet），当请求过大会直接溢出，可以看出漏桶算法能强行限制数据的传输速率。

<br>    
![]({{ site.baseurl }}/img/rate-limiter-package.png)<br>
可以看出，漏桶算法可以强制网络传输的速率，但无法处理突发传输，比如接口请求量在某一时刻突然激增到了十多倍，如果应用接口毫无限制的处理请求返回数据包，很有可能造成整个应用的不可用。这个时候我们就需要使用令牌桶算法，来控制这种突发传输。

- 令牌桶算法：令牌桶算法的原理是系统会以一个恒定的速度往桶里放入令牌，直到到达令牌桶容量，令牌不在增加，而如果请求需要被处理，则需要先从桶里获取一个令牌，当桶里没有令牌可取时，则拒绝服务。
<br>
![]({{ site.baseurl }}/img/rate-limiter-token.png)

## Guava RateLimiter

Guava包含了很多谷歌开源的和核心库，其中就包括了限流工具类 RateLimiter，是基于令牌桶算法实现的。
## 使用
1. 在我们的java项目中pom.xml文件中添加依赖

```xml

<!-- guava -->
<dependency>
	<groupId>com.google.guava</groupId>
	<artifactId>guava</artifactId>
	<version>19.0-rc2</version>
</dependency>
```
2. 在我们需要使用限流的控制类中创建一个reteLimiter对象，用来控制某个接口的请求并发线程数

```java
private final RateLimiter rateLimter = RateLimiter.create(4.0);//每秒钟只有四个请求能处理
```

3. 在接口处添加请求限制

```java
//阻塞请求
rateLimiter.acquire();  //请求RateLimiter, 超过permits会被阻塞
executor.submit(runnable); //提交任务


//非阻塞
if(rateLimiter.tryAcquire()){ //未请求到rateLimiter则立即返回false
    doSomething();
}else{
    doSomethingElse();
}
```
