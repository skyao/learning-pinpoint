Pinpoint Profiler 插件示例
========================

> 注：内容翻译自官方文档 [Plugin Sample](https://github.com/naver/pinpoint-plugin-sample).

可以通过编写 Pinlpoint profiler 插件来扩展 Pinpoint 的profile能力。这个示例项目展示如何编写它。它包含3个模块：

- plugin-sample-target: 目标类库
- plugin-sample-plugin: 示例插件
- plugin-sample-agent: 带有示例插件的agent发行包

# 实现 Profiler 插件

Pinpoint profiler 插件必须提供 [ProfilerPlugin](https://github.com/naver/pinpoint/blob/master/bootstrap-core/src/main/java/com/navercorp/pinpoint/bootstrap/plugin/ProfilerPlugin.java) 和 [TraceMetadataProvider](https://github.com/naver/pinpoint/blob/master/commons/src/main/java/com/navercorp/pinpoint/common/trace/TraceMetadataProvider.java) 的实现. ProfilerPlugin 仅被Pinpoint Agent使用， 而 TraceMetadataProvider 被 Pinpoint Agent, Collector 和 Web使用.

Pinpoint通过 Java [ServiceLoader](https://docs.oracle.com/javase/6/docs/api/java/util/ServiceLoader.html) 机制来装载这些实现。因此 plugin 的 JAR 必须包含两个 provider-configuration 文件：

- META-INF/services/com.navercorp.pinpoint.bootstrap.plugin.ProfilerPlugin
- META-INF/services/com.navercorp.pinpoint.common.trace.TraceMetadataProvider

每个文件应该包含实现类的全限定名。

## TraceMetadataProvider

TraceMetadataProvider 添加 [ServiceTypes](https://github.com/naver/pinpoint/blob/master/commons/src/main/java/com/navercorp/pinpoint/common/trace/ServiceType.java) 和 [AnnotationKeys](https://github.com/naver/pinpoint/blob/master/commons/src/main/java/com/navercorp/pinpoint/common/trace/AnnotationKey.java) 到 Pinpoint.

ServiceType 和 AnnotationKey的编码值必须唯一. 如果编写一个私有插件， 可以使用为私下使用保留的编码值。Pinpoint不会给任何东西分配这些值。否则需要联系 Pinpoint dev team 来为插件分配编码。

- 私下使用的ServiceType编码

    - Server: 1900 ~ 1999
    - DB client: 2900 ~ 2999
    - Cache client: 8999 ~ 8999		(这里的范围貌似有点问题，已经开 [issue](https://github.com/naver/pinpoint-plugin-sample/issues/21) 给pinpoint确认)
    - RPC client: 9900 ~ 9999
    - Others: 7500 ~ 7999

- 私下使用的AnnotaionKey编码

	- 900 ~ 999

## ProfilerPlugin

ProfilerPlugin 添加 [TransformCallbacks](https://github.com/naver/pinpoint/blob/master/bootstrap-core/src/main/java/com/navercorp/pinpoint/bootstrap/instrument/transformer/TransformCallback.java) 到 Pinpoint.

TransformCallback 通过添加interceptors, getters 和/或 fields来转换目标类。可以在plugin-sample-plugin 项目中找到示例代码.

# 集成测试

可以用 [PinointPluginTestSuite](https://github.com/naver/pinpoint/blob/master/test/src/main/java/com/navercorp/pinpoint/test/plugin/PinpointPluginTestSuite.java) (一个JUnit Runner)来运行插件集成测试。它从maven仓库下载需要的依赖并启动一个新的JVM，这个JVM带有Pinpoint profiler agent和依赖。JUnit 测试在这个JVM上执行。

为了运行集成测试，需要一个完整的agent发行包。这也是为什么集成测试放在 plugin-sample-agent 模块中。

在测试中，可以使用 [PluginTestVerifier](https://github.com/naver/pinpoint/blob/master/bootstrap-core/src/main/java/com/navercorp/pinpoint/bootstrap/plugin/test/PluginTestVerifier.java) 来检查跟踪信息/trace是否被正确记录。

## 测试依赖

PinointPluginTestSuite 不使用项目的依赖(配置在pom.xml中). 它使用通过 @Dependency 列出的依赖。以这种方式，可以测试目标类库的多个版本。

依赖这些定义， 你可以指定依赖的版本或者版本范围：

```java
@Dependency({"some.group:some-artifact:1.0", "another.group:another-artifact:2.1-RELEASE"})
@Dependency({"some.group:some-artifact:[1.0,)"})
@Dependency({"some.group:some-artifact:[1.0,1.9]"})
@Dependency({"some.group:some-artifact:[1.0],[2.1],[3.2])"})
```

PinointPluginTestSuite 从本地仓库和maven中央仓库中搜索依赖。可以通过@Repository添加仓库。

## Jvm 版本

可以通过@JvmVersion为测试指定 JVM 版本.

## 应用测试

PinpointPluginTestSuite 不适合用于那些需要通过自己的main class启动的应用。可以扩展 [AbstractPinpointPluginTestSuite](https://github.com/naver/pinpoint/blob/master/test/src/main/java/com/navercorp/pinpoint/test/plugin/AbstractPinpointPluginTestSuite.java) 和相关类型来测试这样的应用。




