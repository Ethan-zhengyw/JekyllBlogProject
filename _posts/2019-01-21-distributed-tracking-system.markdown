---
layout: post
title:  "调用链与分布式追踪系统"
date:   2019-01-21 00:40:07 +0800
categories: blog jekyll SRE Tracking
---
关键词：SRE，调用链，分布式追踪系统

上周的工作内容是开发一个运维需求——对接公有云的分布式追踪系统。服务上线转商用之前，运维能力需要满足商用要求，而使用分布式追踪系统把服务监控起来，是商用运维要求中的必须项，因此有了这个需求。一周下来，代码已经开发得差不多了，还差验证和上线工作，在这之前，先梳理回顾一下。

对接到分布式追踪系统之所以如此重要，是因为使用分布式追踪系统监控服务有以下好处
1. 通过在各个微服务埋点打印的调用链日志，能够把一个请求处理的完整链路端到端地展示出来
2. 能够快速定位到异常环节，如中间在某个环节响应慢或是处理失败
3. 能够根据调用链日志生成服务依赖关系拓扑图，直观地观察到故障点或哪个微服务是系统的瓶颈点所在
4. 甚至能够根据采集的日志进行告警配置

我们的服务主要使用Python和Go语言进行开发，分别获取对应的SDK进行开发即可，整个开发对接上线的流程大致是这样的：
1. 联系接口人沟通埋点方案
2. 使用统一的SDK进行埋点代码开发和测试
3. 整理服务—微服务—节点—日志路径信息，服务—微服务—业务—URL信息。公司的分布式追踪系统的日志采集借用了已有的日志服务的能力，日志服务接收到调用链日志时，会直接转发给分布式追踪系统，因此信息需要分别反馈给日志采集系统和分布式追踪系统在服务节点上安装日志采集Agent
4. 升级微服务，打开打印调用链日志的开关（开始打印调用链日志，与此同时，节点上的日志采集Agent把采集的调用链日志信息上报到日志系统）
5. 类生产环境验证对接效果：本地正常打印调用链日志，日志采集Agent正常上报调用链日志；分布式追踪系统能够检索到服务的Trace信息，能够正常显示微服务依赖关系拓扑图
6. 上线生产环境各个局点

### 埋点开发

#### 哪些地方需要打印调用链日志呢？
和性能调优类似，打点的地方可以包括：HTTP请求服务/客户端、RPC请求服务/客户端、数据库访问、中间件调用、本地方法调用。
> What points should be tracked by default?
I think that for all projects we should include by default 5 kinds of points:
* All HTTP calls - helps to get information about: what HTTP requests were done, duration of calls (latency of service), information about projects involved in request.
* All RPC calls - helps to understand duration of parts of request related to different services in one project. This information is essential to understand which service produce the bottleneck.
* All DB API calls - in some cases slow DB query can produce bottleneck. So it’s quite useful to track how much time request spend in DB layer.
* All driver calls - in case of nova, cinder and others we have vendor drivers. Duration
* ALL SQL requests (turned off by default, because it produce a lot of traffic)

#### 埋点方式
1. 显示调用start和stop方法
2. 函数/类装饰器
3. with代码块
4. 某些框架或依赖库有官方的集成支持，JAVA和Go的支持非常全，而Python的则缺少一些中间件（https://github.com/opentracing-contrib）
5. 听说Go还有一种直接替换Go-SDK就能够完成埋点功能注入的方式...

### 两篇重要的论文
* 《Dapper, a Large-Scale Distributed Systems Tracing Infrastructure》—— 让分布式追踪系统这个领域流行起来
* 《Uncertainty in Aggregate Estimates from Sampled Distributed Traces》—— 关于采样的详细分析

### 标准
Uber发起的OpenTracing现在已经成为云原生计算基金会的一个子项目，[OpenTracing Semantic Specification](https://github.com/opentracing/specification/blob/master/specification.md)定义了数据采集时的数据模型和一些接口，应该算是分布式追踪系统的接口标准了，比较出名的Jaeger和Zipkin都实现了OpenTracing API。

### 业界实践
* 阿里淘宝：EagleEye
* Twitter：Zipkin
* Uber：Jaeger
* AWS：XRays
* Google：Dapper，StakeDriver Trace，OpenCensus
* Golang：AppDash
* 蚂蚁：云图
* 阿里云：谛听，盘古
* 神马：STrace
* OpenStack：osprofiler（原本是用于采集方法调用耗时数据，上报到ceilometer监控计量计费系统，用于性能调优，经过定制修改日志打印格式兼容OpenTracing的话打印出来的就是调用链日志了）

### 参考链接
* [https://blog.csdn.net/yunqiinsight/article/details/80134045](https://blog.csdn.net/yunqiinsight/article/details/80134045)
* [https://yq.aliyun.com/articles/514488](https://yq.aliyun.com/articles/514488)
* [https://www.cnblogs.com/sting2me/p/4393018.html](https://www.cnblogs.com/sting2me/p/4393018.html)
* [https://blog.appoptics.com/intro-to-opentracing-and-opencensus-for-distributed-tracing/](https://blog.appoptics.com/intro-to-opentracing-and-opencensus-for-distributed-tracing/)
* [https://logz.io/blog/zipkin-vs-jaeger/](https://logz.io/blog/zipkin-vs-jaeger/)
* [https://opentracing.io/](https://opentracing.io/)
