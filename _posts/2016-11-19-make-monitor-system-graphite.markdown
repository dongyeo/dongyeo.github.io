---
layout:     post
title:      "用开源产品撸个监控系统（二）——展示存储"
date:       2016-11-18 08:00:00 +0800
author:     "DongYeo"
header-img: "img/post-bg-01.jpg"
tags: ["监控系统","时间序列存储"]
---

## 前言

线上业务越来越繁重，运维体系中看上去最不重要，但实际上是最重要的一环便是监控。快速统计线上业务数据、高效展示监控数据、及时针对业务异常做出针对性的报警、存储监控数据用于大数据计算对在线数据进行预测和异常检测，这些都是监控系统在运维体系中发挥的作用。

根据监控系统的功能，不难分析出，监控系统的极大核心功能分别是：采集、存储、展示、计算、报警。

![monitor-func]({{ site.baseurl }}/img/monitor-func.jpg)

本系列博（bi）客（ji）将记录我使用如下开源工具，搭建一个小巧的监控系统，进而剖析监控系统是如何运作的：

- Logster
- graphite
- storm

## 存储展示工具Graphite

Graphite是一个准企业级的监控系统，很多知名的互联网公司如github/reddit/salesforce都有在使用。说是监控系统，其实只能说是一个处理优化存储时间序列、展示数据的一套框架。其在官网首页的宣传语也是牛逼的不行。→\_→

![graphite-slo]({{ site.baseurl }}/img/graphite.png)

如图所示为Graphite的架构图：

![graphite-slo]({{ site.baseurl }}/img/graphite-archi.png)

### carbon

Carbon是graphite负责接收上报数据的模块，就是负责与上文提到的Logster之类的日志采集客户端交互的模块。所有上报上来的数据，都会在这个模块内进行处理，最后将处理的结果进行存储、聚合、缓存等操作。

Carbon的安装非常的简单，在官网的文档里有详细的介绍，推荐使用pip的方式来安装。

#### carbon-cache:

安装完成之后，我们就可以通过carbon-cache指令运行carbon进程，默认配置下，该进程会监听2003端口的请求。


carbon支持一下三种种通信协议来接受上报的数据：
- Plain text protocol，平文本协议，上文中的Logster的GraphiteOutput模块使用的就是这种平文本协议，通过向目的carbon-cache客户端建立套接字链接，发送平文本协议字符串，来上报监控数据，以使用类似Netcat（nc）命令工具发布(PORT2003)为例：

```bash
echo “<metricPath> <metricValue> <timestamp>”| nc –q0 ${SERVER} ${PORT}
```

- Pickle protocol，多层级元组数据结构(PORT2004)
例：

```
[(path, (timestamp, value)), ...]
payload = pickle.dumps(listOfMetricTuples, protocol=2)
header = struct.pack("!L", len(payload))
message = header + payload
```

- AMQP protocol：Advanced Message Queuing Protocol:

通过该协议从消息队列中间件获取数据，这部分笔者没有实践过，不做详解。

#### carbon-aggregator：

当我们兴致匆匆地启动cache进程，然后启动日志采集工具不断得往cache进程发送数据的时候，你会发现，当在分布式的环境下，实际的数据和采集的数据之间并不是一致的。不用过多分析，大家也很快就等得出答案，我们缺针对数据进行聚合这么一个流程。仔细看carbon的架构，我们发现还有一个carbon-aggregator的模块没用使用。

carbone配置目录下有一个aggregation-rules.conf配置文件，这个配置文件用于配置上报指标的聚合规则，例如：我们采集的指标名称如下：

```
<env>.applications.<app>.<server>.<metric>
```
我们可以在聚合配置文件中如下配置：

```
<env>.applications.<app>.all.requests (60) = sum <env>.applications.<app>.*.requests
<env>.applications.<app>.all.latency (60) = avg <env>.applications.<app>.*.latency
```
当接受如下的数据的时候：

```
prod.applications.apache.www01.requests
prod.applications.apache.www02.requests
prod.applications.apache.www03.requests
prod.applications.apache.www04.requests
prod.applications.apache.www05.requests
```
最后都会以prod.applications.apache.all.request名，***聚合函数*** 结果为值向后端cache进程汇报。


#### carbon-relay:

carbon-relay是carbon实现负载均衡的模块，根据配置将不同的metricName路由到不同的目的地址端口或者一致哈希到不同的目的端口。具体的配置可以在配置模板文件中看具体的说明。

### whisper

whisper是一个时间序列存储的数据库，应用程序可以用create，update和fetch操作获取并操作这些数据。和通常的RRD数据库不同，他可以根据配置将数据存储成不同分辨率的数据，并且高分辨率的数据可以降级成低分辨率的数据。
通过以下两个配置，可以实现一个指标数据从高分辨率到低分辨的转存，这样对于既可以保持最新数据的新鲜度，又可以节省历史数据的存储空间：

```
[apache_busyWorkers]
pattern = ^servers\.www.*\.workers\.busyWorkers*
retentions = 15s:7d,1m:21d,15m:5y

[all_min]
pattern = \.min$
xFilesFactor = 0.1
aggregationMethod = min

```

上面的配置就会将^servers\\.www.\*\\.workers\\.busyWorkers\*的指标以15S一个点存储一周，之后会聚合成1分钟的点存储三周，之后再聚合成15分钟一个点，存储5年。

你可能会问，怎么聚合呢？答案在第二条的配置里，例如，只要名称以min结尾的，那么就会以去最值的方式聚合。

### graphite-web

哼，又臭又长的无脑安装，当解决完所有版本依赖，你就可以跑起这个web应用。

![graphite-slo]({{ site.baseurl }}/img/graphite-web.png)
