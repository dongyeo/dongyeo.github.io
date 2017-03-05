---
layout:     post
title:      "用开源产品撸个监控系统（三）——非完美实现报警系统"
date:       2016-11-18 08:00:00
author:     "DongYeo"
header-img: "img/post-bg-03.jpg"
tags: ["监控系统","日志采集"]
---

## 前言

线上业务越来越繁重，运维体系中看上去最不重要，但实际上是最重要的一环便是监控。快速统计线上业务数据、高效展示监控数据、及时针对业务异常做出针对性的报警、存储监控数据用于大数据计算对在线数据进行预测和异常检测，这些都是监控系统在运维体系中发挥的作用。

根据监控系统的功能，不难分析出，监控系统的极大核心功能分别是：采集、存储、展示、计算、报警。

![monitor-func]({{ site.baseurl }}/img/monitor-func.jpg)

本系列博（bi）客（ji）将记录我使用如下开源工具，搭建一个小巧的监控系统，进而剖析监控系统是如何运作的：

- Logster
- graphite
- storm

## 报警模型

当我们通过前两篇文章的工具将相关业务的指标采集并存储以后，我们需要根据实际的业务特性和关键程度来针对其关键指标配置响应的报警。

例如：某网站的访问PV（200状态码统计）是一个核心的业务指标，在业务正常的情况下，这一指标的曲线是非常的平滑的，当这一指标突然出现下跌，那么，和可能这一业务相关的系统很有可能出现了异常，需要告警来提醒相关人员上线处理异常。

上面那个例子中，我们可以抽象出一个报警规则，200状态码1分钟求和环比下跌XX%，业务出现异常，这个XX%取决于业务正常波动和实际异常的跌幅。那么通常我们有哪些报警的数据计算方式？

报警指标|计算模型|触发类型|特点
---|---|---|---
当前值|阈值比较|边沿触发|适合针对一些稳定的业务指标，例如成功率低于XX%，失败数大于XX
当前值环比|比例阈值比较|边沿触发|适合业务平稳，波动性小，具有一定周期性的业务指标
当前值同比|比例阈值比较|边沿触发|适合周期性强的指标，例如昨日同比、上周同比等
N时间单位聚合|阈值比较|边沿触发|适合指标小，波动大的脉冲式的指标，用于兜底业务，防止N周期内业务量过小的异常**<b style="color:red">以此衍生的还有N时间单位聚合同比</b>**
N时间单位聚合环比|比例阈值比较|边沿触发|适合单位时间波动大，单整体具有一定周期和稳定趋势的业务指标，起到滤波的作用
N时间单位持续|阈值比较|电平触发|适合脉冲、抖动打的业务指标监控，例如持续跌零，持续过高等
周期分解残差|残差的异常检测|算法检测边沿触发|适合周期性极强的业务指标，通过类似于STL的季节分解，将时间序列分解为趋势+周期+残差，最后，对残差使用MWA之类的异常检测算法进行检测


## 技术实现

有了业务指标，有了报警模型，我们就可通过一定的技术手段实现报警逻辑。
报警需要具有非常强的实时性，需要在很短的时间将指标根据报警模型进行计算，并将异常很快的告诉业务方。

开源世界里有很多的实时计算引擎，例如Spark、Storm等，本次我们就使用Storm来作为我们的实时计算引擎，来对我们业务PV进行当前值的监控。

**<b style="color:red">本文提及的所有技术实现都并非最佳方案，所以Strom在这里的使用并没有考虑高并发，大规模监控的情景，后续会有其他监控方案的文章</b>**

![monitor-func]({{ site.baseurl }}/img/storm-logo.png)
先介绍下整个Storm报警的模型的topology:
![monitor-func]({{ site.baseurl }}/img/topology.png)

- ApiStreamingSpout

拓扑的入口ApiStramingSpout是负责向上一章节搭建的graphite-web提高的数据拉取api，拉取我们需要配置报警的指标数据。

```Java
**
 * Created by yeodong on 10/5/16.
 */
public class ApiStreamingSpout extends BaseRichSpout {
    static String STREAMING_APU_URL = "http://localhost:8989/api/foo.json";
    private SpoutOutputCollector collector;
    // this is the task id ,which is corresponded to api
    private static final int taskId = 1;
    private static final SimpleDateFormat simpleDateFormat=new SimpleDateFormat("YYYY-MM-dd HH:mm");
    public void open(Map map, TopologyContext topologyContext, SpoutOutputCollector spoutOutputCollector) {
        this.collector = spoutOutputCollector;
    }

    public void nextTuple() {
        String result = __getData();
        System.out.println("get data at"+ System.currentTimeMillis());
        if(null!=result){
            collector.emit(new Values(taskId,result));
        }
        try {
            Thread.sleep(10000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    public void declareOutputFields(OutputFieldsDeclarer outputFieldsDeclarer) {
        outputFieldsDeclarer.declare(new Fields("taskId","data"));
    }
    private String __getData(){
        String result = null;
        CloseableHttpClient httpclient = HttpClients.createDefault();
        try {
            Date endTime = new Date();
            String endTimeStr = simpleDateFormat.format(endTime);
            Date startTime  = new Date(endTime.getTime()-60*60*1000);
            String startTimeStr = simpleDateFormat.format(startTime);
            //String url = STREAMING_APU_URL+"?startTime="+ URLEncoder.encode(startTimeStr)+"&endTime="+URLEncoder.encode(endTimeStr);
            String url = "http://47.90.65.139:8085/render?target=nginx.http_2xx&from=-1h&format=json";
            //System.out.println("url:"+url);
            HttpGet httpGet = new org.apache.http.client.methods.HttpGet(url);
            CloseableHttpResponse response = httpclient.execute(httpGet);// by the connection manager.
            try {
                //System.out.println(response.getStatusLine().getStatusCode());

                HttpEntity entity1 = response.getEntity();
                byte in[] = new byte[1024];
                StringBuffer sb = new StringBuffer();
                int rl = 0;
                while((rl = entity1.getContent().read(in))>0){
                    sb.append(new String(in,0,rl));
                }
                result = sb.toString();
                // do something useful with the response body
                // and ensure it is fully consumed
               EntityUtils.consume(entity1);
            } finally {
                response.close();
            }
        }catch(Exception e){
            e.printStackTrace();
        }
        return result;
    }
}
```

- JsonParseSpout

用于我们拉取的数据是一个json的字符串，需要通过JsonParseSpout这个类来将字符串解析成Java的pojo，在这里笔者直接取出数据并进行了阈值的判断，并对触发阈值报警的数据，产生一个告警消息，让告警spout进行处理。实际情景下，可以根据不同的数据源，解析成不同的消息，并丢给不同的告警规则处理的spout程序进行处理。

```Java
/**
 * Created by yeodong on 10/10/2016.
 */
public class JsonParserBolt extends BaseRichBolt {

    private OutputCollector collector;
    private final static ConcurrentHashMap<String,Long> lastTimeStamp = new ConcurrentHashMap<String, Long>();
    private static Long THRESHOLD = 500L;
    public void prepare(Map map, TopologyContext topologyContext, OutputCollector outputCollector) {
        this.collector = outputCollector;
    }
    public void execute(Tuple tuple) {
        String jsonStr = tuple.getStringByField("data");
        //Result result = JSON.parseObject(data,Result.class);
        Data  data = JSON.parseObject(jsonStr.substring(1,jsonStr.length()-1),Data.class);
        if(data!=null&&data.datapoints!=null&&!data.datapoints.isEmpty()){
            String rawValue = data.datapoints.get(data.datapoints.size()-1).get(0);
            System.out.println("rawValue"+rawValue);
            Long value = Long.valueOf(rawValue.equals("null")?"0":rawValue.substring(0,rawValue.indexOf('.')));
            Long timeStamp = Long.valueOf(data.datapoints.get(data.datapoints.size()-1).get(1));
            Long lastTime = lastTimeStamp.get(data.target);
            if(value <THRESHOLD &&(lastTime==null || !lastTime.equals(timeStamp))){
                collector.emit(new Values(data.target,"PV lower than "+THRESHOLD,timeStamp));
            }
            lastTimeStamp.put(data.target,timeStamp);
        }
    }

    public void declareOutputFields(OutputFieldsDeclarer outputFieldsDeclarer) {
        outputFieldsDeclarer.declare(new Fields("target","message","timeStamp"));

    }
}

class Data{
    public String target;
    public List<List<String>> datapoints;

```

- AlertSpout

用于将上游经过JsonParseSpout处理的上游产生的告警信息，并调用告警模块通知先关的人员，由于是示范程序，只在终端进行的告警的打印。


```Java
/**
 * Created by yeodong on 13/10/2016.
 */
public class AlertBolt extends BaseRichBolt {
    private OutputCollector collector;
    public void prepare(Map map, TopologyContext topologyContext, OutputCollector outputCollector) {
        this.collector = outputCollector;
    }
    public void execute(Tuple tuple) {
        String target = tuple.getStringByField("target");
        String message = tuple.getStringByField("message");
        Long timeStamp = tuple.getLongByField("timeStamp");

        System.out.println("===================================================");
        System.out.println(target+" at "+timeStamp+":"+message);
        System.out.println("===================================================");

    }

    public void declareOutputFields(OutputFieldsDeclarer outputFieldsDeclarer) {

    }
}
```


## 总结

至此，本人在阿里监控中心新人的监控系统的相关的学习就算完了，所有的东西都不太成熟，主要的目的是学习监控相关的知识，其中也带了很多自己意淫的成分，业内有很多相关的成熟的监控解决方案，这是我后续将会继续学习的方向。如果你看到我写的这几篇学习摸索的笔记，可以给我留言，告诉我其中的不足。
