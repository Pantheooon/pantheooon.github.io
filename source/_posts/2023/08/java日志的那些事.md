---
layout: post
title: java日志的那些事
date: 2023-08-17
tags: ["elk","kafka","splunk","开发","日志"]
---

# logback or log4j or slf4j

很多人对这些组件感到很熟悉,但是又有点陌生,这里主要介绍这几个组件的来历.
<!--more-->

在`javase1.4`的时候,`sun`公司发布了一个`JUL(Java Util Logging)`日志组件,这个组件存在一些问题,比如配置缺乏灵活性,功能有限.甚至还有性能问题,俄罗斯程序员`Ceki Gülcü`设计了一款日志组件`log4j`,它完美解决了`JUL`的上述问题,后来这哥们就把`log4j`捐给了`apache`,在这段时间里面`Ceki Gülcü`对`apache`的管理方式颇有言辞,就另起炉灶创造出来了`logback`,同时`logback`和`log4j`之间的api有差异性,又创造出了`slf4j`,所以`slf4j`最早是为了解决`logback`和`log4j`兼容性而开发的,再后来`apache`因为`logback`受到冲击,停止维护`log4j`,转而开发新的日志组件`log4j2`,至此故事告一段落.

`logback`和`log4j`可以看成是一个项目组件中真正在干活的,而`slf4j`则是他们的接口,假设一下,抽象出这一层可以屏蔽掉底层接口的细节,从而提供统一接口规范,也就是所谓的`facade模式`.

# logback的配置

这里以logback为例,解释下logback.xml中常见的配置概念:

*   layout:`layout`的作用就是把输出的日志按照指定格式`pattern`来输出,`pattern`里面会有很多占位符,比如`%d{yyyy-MM-dd HH:mm:ss.SSS}`,所以layout第一件事情要做的事情就是解析设置的`pattern`按照`'`分割,将他们变成一个个的token,在找到相应的`converter`或者从`MDC`从获取值把它填充进去.
*   MDC:`MDC`就是一个`threadlocal`的map,主要作用就是塞值,然后配置`layout`,输出相应的日志.
*   encoder:日志再经过`layout`之后.会返回一个`String`,也就是最终输出的日志,`encoder`作用就是把它变成`byte`,方便后续的传输
*   converter:这个就是一个接口,`ClassicConverter`用于获取token的值,比如`pattern`是这样的,`%d{yyyy-MM-dd HH:mm:ss.SSS}'${appDev}'${appName}'%level'%hostName'%thread'%logger{50}'%tid'%X{TRACE-ID}'%msg%n`,这里面有个`hostname`的token,他的值会从这个定义的类中获取`<conversionRule conversionWord="hostName" converterClass="com.pantheon.logback.IpConvert"/>`
*   appender:`appender`其实就是决定了`encoder`之后的`byte`要输出到哪里,可以输出到`kafka`,也可以输出到`console`,也可以输出到磁盘上.

# 日志采集和入库

## 输出到磁盘

![输出到磁盘](20230817130714.png)

流程如下

1.  应用输出日志到磁盘
2.  filebeat实时采集日志将日志输出到kafka
3.  在有logstah监听kafka,将日志转换成json格式,写入到es

这里加了一个kafka的组件,主要用作写es的缓冲层,避免因为日志量过大,而将es打垮的可能性.

输出到磁盘的好处在于:日志有多份,防止因为采集丢失日志而无法排查问题,当然缺点也很明显,因为磁盘有容量大小,一旦磁盘满了就会导致应用宕机.此外,如果应用是部署在k8s里面,磁盘就不能是pod里面的磁盘,要做pv,还要用deamonset来采集pv里面的日志,比较麻烦.

## 输出到kafka

![输出到kafka](20230817131454.png)

上面这个流程去掉了输出到磁盘这一部分,而由应用直接将日志输出kafka,避免了上述可能出现的问题,但是它也有缺点,一旦项目输出到日志不能被kafka收集,比如网络问题,程序假死等,那对排查问题带来很多不便.

# schema on read or schema on write

这个概念来源于`splunk`,`splunk`是一家著名的日志解决方案供应商,同名产品`splunk`提供了日志收集以及索引,实时监控告警,数据分析和可视化等一些列强大的功能,可以理解为是elk的增强版.

首先`schema`这个概念,其实就是说日志格式问题,比如我们用elk来采集,输出的日志是这样:

    2023-08-17 10:48:27.349'test'test-order'INFO'10.2.11.15'host.docker.internal'main'com.alibaba.nacos.client.config.impl.ClientWorker'N/A''这里是输出的日志

但是我们在写入es的时候会变成这样

    {
        "message": "这里是输出的日志",
        "@timestamp": "2023-08-17 10:48:27.349",
        "appname": "test-order",
        "ip": "172.25.5.152",
        "hostname": "test-order-java-77b6cc8cc5-gp9vp",
        "tid": "3008dc5972434bf6a5a5401e970c69e9",
        "skyid": "TID:N/A",
        "env": "test",
        "thead": "http-nio-8080-exec-5",
        "class": "com.alibaba.nacos.client.config.impl.ClientWorker",
        "loglevel": "INFO",
        "time": "2023-08-17 13:27:42.296"
    }

在使用elk的时候,elk对整个log进行了清洗,,而对`message`本身却无能为力,因为它非结构化,只是一段文本,所以想要对`message`做很强大的聚合和分析,这个时候elk就捉襟见肘,这种在写入就将数据清洗搞定的方式就叫做`schema on write`,这种方式类似于跑hadoop任务,事先需要将想要的数据按照格式进行整理,再来跑hive脚本,缺点很明显,很不灵活.

而`splunk`则对数据分析提供了更多的功能,比如我有一个日志是这样的`login success,userId:8u1123mmuujaa`,想要根据这一类日志找出一天重复登录成功的用户,用es来实现会变得极其复杂,而`splunk`提供了类似`pipline`的语法,去抽取其中的数据,将其转化成结构化的结构,在将编写的语句变成job,分发到不同的节点上运行,这种在运行时去确定数据结构的做法叫做`schema on read`,这种做法在实际运用中非常方便,因为不需要事先对数据进行清洗,只需要调整我们的脚本,就能得到想要的数据.
除了`splunk`还有类似的产品,比如[炎凰数据](https://www.51cto.com/article/755320.html),sls,Azure data explore,日志易等等,当然他们也有一个缺点:价格很昂贵.

# dashboard

我们再对日志进行聚合分析后,需要将其展示出来,常见的展示图有这几种:

*   饼状图
*   折现图
*   99线,95线:这个在对系统做性能分析的时候很有用,一眼就能看出长尾数据
*   环比图:可以是同一个事务在不同一个时期对比,也可以是不同的事物在同一时期之间的对比,也很直观
`ELK`中提供了kibana作为日志分析聚合展示的dashboard,当然也有其他的方案比如`granfa`.

# alert

基于日志聚合的结果作为告警的方式,原理就是给定一个固定的时间窗口,在时间窗口中运行脚本最终得到一个值,把这个值和阈值对比,大于或者小于则发出告警,更多关于alert的可以参考这篇文章:[https://blog.pantheon.press/?p=135](https://blog.pantheon.press/?p=135) ,在`elk`的解决方案中`alert`属于白金版的功能,所以到目前为止,很少看到用kibana来做alert的.