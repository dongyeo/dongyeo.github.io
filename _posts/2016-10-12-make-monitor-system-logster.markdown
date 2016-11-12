---
layout:     post
title:      "用开源产品撸个监控系统(1)"
date:       2016-10-12 12:00:00
author:     "DongYeo"
header-img: "img/post-bg-02.jpg"
---


## 前言

线上业务越来越繁重，运维体系中看上去最不重要，但实际上是最重要的一环便是监控。快速统计线上业务数据、高效展示监控数据、及时针对业务异常做出针对性的报警、存储监控数据用于大数据计算对在线数据进行预测和异常检测，这些都是监控系统在运维体系中发挥的作用。

根据监控系统的功能，不难分析出，监控系统的极大核心功能分别是：采集、存储、展示、计算、报警。

![monitor-func]({{ site.baseurl }}/img/monitor-func.jpg)

本系列博（bi）客（ji）将记录我使用如下开源工具，搭建一个小巧的监控系统，进而剖析监控系统是如何运作的：

- Logster
- graphite
- storm

---

## 采集工具 Logster

Logster是一个开源的日志采集工具，主要的功能是增量地读取日志文件，并根据预定的规则解析，生成监控数据，并根据相关监控系统的通信协议，上报给监控系统。[github link](https://github.com/etsy/logster)

![logster]({{ site.baseurl }}/img/logster.jpg)

如上图所示，Logster由三部分组成：Tailer/Parser/Ouput

- Tailer

  Tailer是Logter增量读取日志文件的组件，默认情况下，它使用logtail组件，我们也可以使用pygtail。具体安装可以github页面看到，非常简单。下面是Tailer的基类，他定义了Tailer来个主要的接口：create_statefile以及ireadlines。

  ``` python
  class Tailer(object):
    """ Base class for tailer implementations """
    def __init__(self, logfile, statefile, options, logger):
        self.logfile = logfile
        self.statefile = statefile
        self.options = options
        self.logger = logger

    def create_statefile(self):
        """ Create a statefile, with the offset of the end of the log file.
        Override if your tailer implementation can do this more efficiently
        """
        for _ in self.ireadlines():
            pass

    def ireadlines(self):
        """ Return a generator over lines in the logfile, updating the
        statefile when the generator is exhausted
        """
        raise NotImplementedError()
  ```
  看两个接口的名字，我们大概也能猜出Tailer的功能：生成记录文件，记录日志读取位置/读取日志文件行。

- Parser

  Parser是处理日志文件，生成监控数据的组件。用户可以根据自己的需求，编写自己的组件。下面是Parser的基类LosgsterParser:

  ```python
  class LogsterParser(object):
    """Base class for logster parsers"""
    def parse_line(self, line):
        """Take a line and do any parsing we need to do. Required for parsers"""
        raise RuntimeError("Implement me!")

    def get_state(self, duration):
        """Run any calculations needed and return list of metric objects"""
        raise RuntimeError("Implement me!")
  ```

  根据基类定义的接口，我们可以非常清楚看出，parser的功能是统计解析行的信息，返回监控数据，下面我们来看一下一个处理access.log日志文件的parser，他的功能是根据正则规则匹配统计出Nginx各类状态码的数量，返回一段时间内各类错误量（时间间隔用户可以自己定义）：

  ```python
  import time
  import re

  from logster.logster_helper import MetricObject, LogsterParser
  from logster.logster_helper import LogsterParsingException

  class SampleLogster(LogsterParser):

      def __init__(self, option_string=None):
          '''Initialize any data structures or variables needed for keeping track
          of the tasty bits we find in the log we are parsing.'''
          self.http_1xx = 0
          self.http_2xx = 0
          self.http_3xx = 0
          self.http_4xx = 0
          self.http_5xx = 0

          # Regular expression for matching lines we are interested in, and capturing
          # fields from the line (in this case, http_status_code).
          self.reg = re.compile('.*HTTP/1.\d\" (?P<http_status_code>\d{3}) .*')


      def parse_line(self, line):
          '''This function should digest the contents of one line at a time, updating
          object's state variables. Takes a single argument, the line to be parsed.'''

          try:
              # Apply regular expression to each line and extract interesting bits.
              regMatch = self.reg.match(line)

              if regMatch:
                  linebits = regMatch.groupdict()
                  status = int(linebits['http_status_code'])

                  if (status < 200):
                      self.http_1xx += 1
                  elif (status < 300):
                      self.http_2xx += 1
                  elif (status < 400):
                      self.http_3xx += 1
                  elif (status < 500):
                      self.http_4xx += 1
                  else:
                      self.http_5xx += 1

              else:
                  raise LogsterParsingException("regmatch failed to match")

          except Exception as e:
              raise LogsterParsingException("regmatch or contents failed with %s" % e)


      def get_state(self, duration):
          '''Run any necessary calculations on the data collected from the logs
          and return a list of metric objects.'''

          # Return a list of metrics objects
          return [
              MetricObject("http_1xx", (self.http_1xx), "1xx status code count"),
              MetricObject("http_2xx", (self.http_2xx), "2xx status code count"),
              MetricObject("http_3xx", (self.http_3xx), "3xx status code count"),
              MetricObject("http_4xx", (self.http_4xx), "4xx status code count"),
              MetricObject("http_5xx", (self.http_5xx), "5xx status code count"),
          ]
  ```

- Output

  Output是跟其他系统打交道用的，比如建立套接字连接，向Graphite这类的存储和展示系统上报数据。老规矩，我们先看下基类LogsterOutput：

  ```python
  class LogsterOutput(object):
    """ Base class for logster outputs"""
    def __init__(self, parser, options, logger):
        self.options = options
        self.logger = logger
        self.dry_run = options.dry_run

    def get_metric_name(self, metric, separator="."):
        """ Convenience method for contructing metric names
             Takes into account any supplied prefix/suffix options"""
        metric_name = metric.name
        if self.options.metric_prefix:
            metric_name = self.options.metric_prefix + separator + metric_name
        if self.options.metric_suffix:
            metric_name = metric_name + separator + self.options.metric_suffix
        return metric_name

    def submit(self, metrics):
        """Send metrics to the specific output"""
        raise RuntimeError("Implement me!")
  ```
  output只需实现一个submit接口，我们来看下GraphiteOutput是如何实现的：

  ```python
  def submit(self, metrics):

        if (not self.dry_run):
            host = self.graphite_host.split(':')

            if self.graphite_protocol == 'udp':
                s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
            else:
                s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

            s.connect((host[0], int(host[1])))

        try:
            for metric in metrics:
                metric_name = self.get_metric_name(metric)

                # Spaces in graphite metric names will cause failure
                if ' ' in metric_name:
                    self.logger.error('Invalid metric name, spaces not allowed')
                    return

                metric_string = "%s %s %s" % (metric_name, metric.value, metric.timestamp)
                self.logger.debug("Submitting Graphite metric: %s" % metric_string)

                if (not self.dry_run):
                    s.sendall(("%s\n" % metric_string).encode('ascii'))
                else:
                    print("%s %s" % (self.graphite_host, metric_string))
        finally:
            if (not self.dry_run):
                s.close()
  ```
  看代码易得，根据指令的host等信息，output建立了一个和graphite的socket链接，并将监控数据按照"%s %s %s" % (metric_name, metric.value, metric.timestamp)的格式，使用graphite的平文本协议，将监控数据上报给了graphite系统。

## 总结

在弄清楚了Logster的工作原理之后，我们就可以使用如下的指令调用，向上上报监控数据啦：

> $ sudo /usr/bin/logster --tailer=pygtail --output=graphite --graphite-host=graphite.example.com:2003 SampleLogster /usr/local/nginx/log/access.log
