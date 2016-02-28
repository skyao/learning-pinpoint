安装指南
=======

> 注：内容翻译自 官方文档 [Installation Guide](https://github.com/naver/pinpoint/blob/master/doc/installation.md)

# 安装

为了搭建自有的Pinpoint实例，需要运行这些组件：

- HBase (用于存储)
- Pinpoint Collector (部署在web容器中)
- Pinpoint Web (部署在web容器中)
- Pinpoint Agent (附加到 java 应用来做采样/profile)

如果要尝试简单的快速开始项目，请参考 [quick-start guide](https://github.com/naver/pinpoint/blob/master/quickstart/README.md)

# 快速概述

1. HBase
    - 搭建 HBase 集群 - [Apache HBase](http://hbase.apache.org/)
    - 创建 HBase Schemas - 在hbase shell上执行 /scripts/hbase-create.hbase

2. 构建Pinpoint (仅当从源代码开始构建时需要)

    - Clone Pinpoint - git clone $PINPOINT_GIT_REPOSITORY
    - 设置 JAVA_6_HOME 环境变量到 JDK 6 home 目录.
    - 设置 JAVA_7_HOME 环境变量到 JDK 7+ home 目录.
    - 在pinpoint 根目录运行 mvn install -Dmaven.test.skip=true

3. Pinpoint Collector

    - 部署 pinpoint-collector-$VERSION.war 到web容器
    - 配置 pinpoint-collector.properties, hbase.properties.
    - 启动容器

4. Pinpoint Web

    - 部署 pinpoint-web-$VERSION.war 到web容器
    - 配置 pinpoint-web.properties, hbase.properties.
    - 启动容器

5. Pinpoint Agent

    - 解压/移动 pinpoint-agent/ 到一个方便的位置 ($AGENT_PATH).
    - 设置 -javaagent:$AGENT_PATH/pinpoint-bootstrap-$VERSION.jar JVM 参数以便将agent附加到java应用
    - 设置 -Dpinpoint.agentId 和 -Dpinpoint.applicationName 命令行参数
    - 用上面的设置启动 java 应用

# HBase

Pinpoint 为collector和web使用 HBase 作为它的存储后端 .

为了搭建自己的集群， 参考 [Hbase网站](http://hbase.apache.org/)。下面给出的是HBase 兼容性表单：

| Pinpoint Version | HBase 0.94.x | HBase 0.98.x | HBase 1.0.x | HBase 1.1.x |
|--------|--------|--------|--------|--------|
| 1.0.x |  yes  |   no    |    no    |     no   |
| 1.1.x |  no  |   not tested    |    yes    |     not tested   |
| 1.5.x |  no  |   not tested    |    yes    |     not tested   |

一旦搭建并运行好HBase，请确保Collector和Web被正确的配置并能够连接到HBase。

## 创建 Schema

有两个脚本可以为pinpoint创建表：hbase-create.hbase 和 hbase-create-snappy.hbase.使用 hbase-create-snappy.hbase 来实现 snappy 压缩 (需要 [snappy](http://code.google.com/p/snappy)), 其他情况使用 hbase-create.hbase.

为了运行这些脚本， 在HBase shell 中如下执行：

```bash
$HBASE_HOME/bin/hbase shell hbase-create.hbase
```

脚本的完整列表见 [这里](https://github.com/naver/pinpoint/blob/master/scripts).

# 构建Pinpoint

有两个选择：

1. 从 [最新的发布](https://github.com/naver/pinpoint/releases/latest) 中下载构建结果并跳过构建过程， 推荐!
2. 从Git clone中手工构建

为了手工构建，必须满足下列要求：

- 安装有JDK 6
- 安装有JDK 7+
- 安装有Maven 3.2.x+
- JAVA_6_HOME 环境变量设置为 JDK 6 home 目录
- JAVA_7_HOME 环境变量设置为 JDK 7+ home 目录

需要JDK 7+ 和 JAVA_7_HOME 环境变量来构建 profiler-optional. 更多关于 optional package 的信息，请看 [这里](https://github.com/naver/pinpoint/blob/master/profiler-optional/README.md).

另外, 为了运行Pinpoint的每个组件所需要的Java 版本列举在下面:

| Pinpoint Version | Agent | Collector | Web |
|--------|--------|--------|--------|
| 1.0.x |    6+    |    6+    |    6+    |
| 1.1.x |    6+    |    7+    |    7+    |
| 1.5.x |    6+    |    7+    |    7+    |

如果上面的要求满足了，就可以简单运行下面的命令：

	mvn install -Dmaven.test.skip=true

安装指南后面将使用 $PINPOINT_PATH 来引用 pinpoint home目录的全路径。

不管那种方法，应该以后面章节中提到的文件和目录告终。

# Pinpoint Collector

需要有下面的war文件来部署到web容器中：

	pinpoint-collector-$VERSION.war

如果手工构建，这个文件的路径会是 $PINPOINT_PATH/collector/target/pinpoint-collector-$VERSION.war。

## 安装

由于Pinpoint Collector 被打包为可部署的war文件，可以像部署其他web应用一样部署到web容器。

## 配置

Pinpoint Collector 有 2 个配置文件: pinpoint-collector.properties 和 hbase.properties.

- pinpoint-collector.properties： 包含colletor的配置。和agent的配置项一起检查下面的值：

    - collector.tcpListenPort (agent中是 profiler.collector.tcp.port - 默认: 9994)
    - collector.udpStatListenPort (agent中是 profiler.collector.stat.port - 默认: 9995)
    - collector.udpSpanListenPort (agent中是 profiler.collector.span.port - 默认: 9996)

- hbase.properties - 包含连接到HBase的配置

    - hbase.client.host (默认: localhost)
    - hbase.client.port (默认: 2181)

这些配置文件在war文件下的 WEB-INF/classes/ 目录.

可以在这里看一下默认配置文件: [pinpoint-collector.properties](https://github.com/naver/pinpoint/blob/master/collector/src/main/resources/pinpoint-collector.properties), [hbase.properties](https://github.com/naver/pinpoint/blob/master/collector/src/main/resources/hbase.properties)








