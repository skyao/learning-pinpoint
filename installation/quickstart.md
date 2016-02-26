快速开始
======

> 注：内容翻译自 [官方quick start文档](https://github.com/naver/pinpoint/blob/master/quickstart/README.md)，增加了少量补充和说明。

Pinpoint有三个主要组件(collector, web, agent)，并使用HBase作为存储。Collector和Web被打包为单个war文件，而agent被打包以便可以作为java agent附加到应用。

Pinpoint quickstart 为agent提供一个示例TestApp， 并使用tomcat maven插件来启动所有三个组件。

# 要求

为了构建pinpoint， 下列要求必须满足：

- 安装有JDK 6
- 安装有JDK 8
- 安装有Maven 3.2.x+
- 环境变量JAVA_6_HOME 设置为 JDK 6 home 目录
- 环境变量JAVA_7_HOME 设置为 JDK 7+ home 目录
- 环境变量JAVA_8_HOME 设置为 JDK 8+ home 目录

QuickStart 支持 Linux, OSX, 和 Windows.

> 注：没有说要不要安装jdk7，顺便一起安装吧。下面是/etc/profile的设置：

```bash
# use by pinpoint compile
export JAVA_6_HOME=/usr/lib/jvm/java-6-oracle/
export JAVA_7_HOME=/usr/lib/jvm/java-7-oracle/
export JAVA_8_HOME=/usr/lib/jvm/java-8-oracle/
```

# 开始

使用 git clone https://github.com/naver/pinpoint.git 下载pinpoint或者将项目作为zip文件打包下载然后解压。

使用maven安装pinpoint并运行 mvn install -Dmaven.test.skip=true

> 注:需要执行的命令如下：
>
```bash
git clone https://github.com/naver/pinpoint.git
cd pinpoint
mvn install -Dmaven.test.skip=true
```

# 安装并启动HBase

下面脚本从 [Apache 下载站点](http://apache.mirror.cdnetworks.com/hbase/) 单独下载HBase.

    对于Windows, 需要从Apache下载站点手工下载HBase.
    下载 HBase-1.0.1-bin.tar.gz 并解压缩.
    重命名目录为 hbase 以便使得最终hbase目录看上去是 quickstart\hbase\hbase.
	另外注意通过相应的.cmd文件来运行脚本。

下载并启动 - 运行 quickstart/bin/start-hbase.sh

初始化表 - 运行 quickstart/bin/init-hbase.sh

> 补充：这里面有两个地方要特别注意
>
> 1. 如果手工下载HBase，按照上面要求解压并重命名为路径quickstart\hbase\hbase。启动时会出错，因为start-hbase.sh文件中hbase配置的路径是"HBASE_VERSION=hbase-1.0.1"，需要手工修改为"HBASE_VERSION=hbase"
> 2. init-hbase.sh不仅仅第一次运行时需要执行，以后再启动quickstart时，也需要执行，否则collector和web启动时会始终无法成功最后180秒超时报错退出。再多执行一次init-hbase.sh就可以正常启动。

# 启动pinpoint守护进程

Collector - 运行 quickstart/bin/start-collector.sh

Web UI - 运行 quickstart/bin/start-web.sh

TestApp - 运行 quickstart/bin/start-testapp.sh

> 注：这三个脚本启动后，用ctrl + c可以退出控制台，此时后台进程还在，但是会看不到日志。因此建议这三个脚本分别在三个不同的终端中执行，这样就可以方便查看每个组件的日志信息。

一旦启动脚本完成，tomcat 日志的最后10行显示在控制台：

- Collector

	![](https://github.com/naver/pinpoint/raw/master/doc/img/ss_quickstart-collector-log.png)
	> 注：如果启动不起来，总是打印"starting pinpoint-quickstart-web \*\*/180 (close wait limit)"，最后180秒超时失败。请尝试再次执行一遍"init-hbase.sh".

- Web UI

	![](https://github.com/naver/pinpoint/raw/master/doc/img/ss_quickstart-web-log.png)

- TestApp

	![](https://github.com/naver/pinpoint/raw/master/doc/img/ss_quickstart-testapp-log.png)

# 检查状态

一旦HBase和三个守护进程在运行，可以访问下面地址来测试自己的pinpoint实例。

Web UI - http://localhost:28080
TestApp - http://localhost:28081

可以通过使用TestApp UI来产生追踪数据给pinpoint， 并使用pinpoint Web UI来检查。TestApp作为test-agent注册在TESTAPP下。

# 停止

HBase - 运行 quickstart/bin/stop-hbase.sh

Collector - 运行 quickstart/bin/stop-collector.sh

Web UI - 运行 quickstart/bin/stop-web.sh

TestApp - 运行 quickstart/bin/stop-testapp.sh

# 额外

pinpoint Web使用mysql来持久化用户/用户组，和警告配置。

而Quickstart使用MockDAO来减少内存使用。

此外如果想使用mysql来执行Quickstart， 请参考Pinpoint [Web's applicationContext-dao-config.xml](https://github.com/naver/pinpoint/blob/master/web/src/main/resources/applicationContext-dao-config.xml) , [jdbc.properties](https://github.com/naver/pinpoint/blob/master/web/src/main/resources/jdbc.properties).

此外，如果想开启告警，需要实现额外逻辑。请参考这个 [链接](https://github.com/naver/pinpoint/blob/master/doc/alarm.md)。

> 注： 上面这个Alarm文档的中文翻译版本在 [这里](../alarm/alarm.md)

