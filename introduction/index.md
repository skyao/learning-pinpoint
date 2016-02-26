Pinpoint
===========

> 翻译自 Pinpoint 的 [github 首页内容](https://github.com/naver/pinpoint)

# 介绍

Pinpoint是一个开源的 APM (Application Performance Management/应用性能管理)工具，用于基于java的大规模分布式系统。

仿照 [Google Dapper](http://research.google.com/pubs/pub36356.html) , Pinpoint 通过跟踪分布式应用之间的调用来提供解决方案，以帮助分析系统的总体结构和内部模块之间如何相互联系.

> 注：对于各个模块之间的通讯英文原文中用的是transaction一词，但是我觉得如果翻译为"事务"容易引起误解，所以替换为"交互"或者"调用"这种比较直白的字眼。

在使用上力图简单高效：

- 安装agent，不需要修改哪怕一行代码
- 最小化性能损失

# 概述

如今的服务通常由很多不同模块组成，他们之间相互调用并通过API调用外部服务。每个交互是如何被执行的通常是一个黑盒。Pinpoint跟踪这些模块之间的调用流并提供清晰的视图来定位问题区域和潜在瓶颈。

- 服务器地图(ServerMap)

	通过可视化分布式系统的模块和他们之间的相互联系来理解系统拓扑。点击某个节点会展示这个模块的详情，比如它当前的状态和请求数量。

- 实时活动线程图表(Realtime Active Thread Chart)

	实时监控应用内部的活动线程。

- 请求/应答分布图表(Request/Response Scatter Chart)

	长期可视化请求数量和应答模式来定位潜在问题。通过在图表上拉拽可以选择请求查看更多的详细信息。

	![](./images/ss_server-map.png)

- 调用栈(CallStack)

	在分布式环境中为每个调用生成代码级别的可视图，在单个视图中定位瓶颈和失败点。

	![](./images/ss_call-stack.png)

- 巡查(Inspector)

	查看应用上的其他详细信息，比如CPU使用率，内存/垃圾回收，TPS，和JVM参数。

    ![](./images/ss_inspector.png)

# 架构

![](./images/pinpoint-architecture.png)

# 支持模块

- JDK 6+
- Tomcat 6/7/8, Jetty 8/9
- Spring, Spring Boot
- Apache HTTP Client 3.x/4.x, JDK HttpConnector, GoogleHttpClient, OkHttpClient, NingAsyncHttpClient
- Thrift Client, Thrift Service
- MySQL, Oracle, MSSQL, CUBRID, DBCP, POSTGRESQL
- Arcus, Memcached, Redis
- iBATIS, MyBatis
- gson, Jackson, Json Lib
- log4j, Logback

