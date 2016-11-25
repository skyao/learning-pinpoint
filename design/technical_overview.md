Pinpoint技术概述
===============

> 注： 内容翻译自官方文档 [Technical Overview Of Pinpoint](https://github.com/naver/pinpoint/wiki/Technical-Overview-Of-Pinpoint), 内容有点长，但是强烈推荐阅读！基本上这是目前pinpoint唯一的一份详细介绍设计和实现的资料。

Pinpoint是一个分析大型分布式系统的平台，提供解决方案来处理海量跟踪数据。2012年七月开始开发，2015年1月9日作为开源项目启动。

本文将介绍Pinpoint： 什么促使我们开始搭建它， 用了什么技术， 还有Pinpoint agent是如何优化的。

# 开始动机 & Pinpoint特点

和如今相比， 过去的因特网的用户数量相对较小，而因特网服务的架构也没那么复杂。web服务通常使用两层(web 服务器和数据库)或三层（web服务器，应用服务器和数据库）架构。然而在如今，随着互联网的成长，需要支持大量的并发连接，并且需要将功能和服务有机结合，导致更加复杂的软件栈组合。更确切地说，比三层层次更多的n层架构变得更加普遍。SOA或者微服务架构成为现实。

系统的复杂度因此提升。系统越复杂，越难解决问题，例如系统失败或者性能问题。在三层架构中找到解决方案还不是太难，仅仅需要分析3个组件比如web服务器，应用服务器和数据库，而服务器数量也不多。但是，如果问题发生在n层架构中，就需要调查大量的组件和服务器。另一个问题是仅仅分析单个组件很难看到大局;当发生一个低可见度的问题时，系统复杂度越高，就需要更长的时间来查找原因。最糟糕的是，某些情况下我们甚至可能无法查找出来。

这样的问题也发生在NAVER的系统中。使用了大量工具如应用性能管理(APM)但是还不足以有效处理问题。因此我们最终决定为n层架构开发新的跟踪平台，为n层架构的系统提供解决方案。

Pinpoint, 2012年七月开始开发，在2015年1月作为一个开源项目启动, 是一个为大型分布式系统服务的n层架构跟踪平台。 Pinpoint的特点如下:

- 分布式事务跟踪，跟踪跨分布式应用的消息
- 自动检测应用拓扑，帮助你搞清楚应用的架构
- 水平扩展以便支持大规模服务器集群
- 提供代码级别的可见性以便轻松定位失败点和瓶颈
- 使用字节码增强技术，添加新功能而无需修改代码

本文将讲述Pinpoint的技术，例如事务跟踪和字节码增强。还会解释应用在pinpoint agent中的优化方法，agent修改字节码并记录性能数据。

# 分布式事务跟踪，基于google Dapper

pinpoint跟踪单个事务中的分布式请求，基于google Dapper。

## 在Google Dapper中分布式事务追踪是如何工作的

当一个消息从Node1发送到Node2(见图1)时，分布式追踪系统的核心是在分布式系统中识别在Node1中处理的消息和在Node2中出的消息之间的关系。

![](https://github.com/naver/pinpoint/raw/master/doc/img/td_figure1.png)

图1. 分布式系统中的消息关系

问题在于无法在消息之间识别关系。例如，我们无法识别从Node1发送的第N个消息和Node2接收到的N'消息之间的关系。换句话说，当Node1发送完第X个消息时，是无法在Node2接收到的N的消息里面识别出第X个消息的。有一种方式试图在TCP或者操作系统层面追踪消息。但是，实现很复杂而且性能低下，而且需要为每个协议单独实现。另外，很难精确追踪消息。

不过，Google dapper实现了一个简单的解决方案来解决这个问题。这个解决方案通过在发送消息时添加应用级别的标签作为消息之间的关联。例如，在HTTP请求中的HTTP header中为消息添加一个标签信息并使用这个标签跟踪消息。

> Google's Dapper
>
> 关于Google Dapper的更多信息, 请见 "[Dapper, a Large-Scale Distributed Systems Tracing Infrastructure.](http://research.google.com/pubs/pub36356.html)"

Pinpoint基于google dapper的跟踪技术,但是已经修改为在调用的header中添加应用级别标签数据以便在远程调用中跟踪分布式事务。标签数据由多个key组成，定义为TraceId。

# Pinpoint中的数据结构

Pinpoint中，核心数据结构由Span, Trace, 和 TraceId组成。

- Span: RPC (远程过程调用/remote procedure call)跟踪的基本单元; 当一个RPC调用到达时指示工作已经处理完成并包含跟踪数据。为了确保代码级别的可见性，Span拥有带SpanEvent标签的子结构作为数据结构。每个Span包含一个TraceId。
- Trace: 多个Span的集合; 由关联的RPC (Spans)组成. 在同一个trace中的span共享相同的TransactionId。Trace通过SpanId和ParentSpanId整理为继承树结构.
- TraceId: 由 TransactionId, SpanId, 和 ParentSpanId 组成的key的集合. TransactionId 指明消息ID，而SpanId 和 ParentSpanId 表示RPC的父-子关系。
    - TransactionId (TxId): 在分布式系统间单个事务发送/接收的消息的ID; 必须跨整个服务器集群做到全局唯一.
    - SpanId: 当收到RPC消息时处理的工作的ID; 在RPC请求到达节点时生成。
    - ParentSpanId (pSpanId): 发起RPC调用的父span的SpanId. 如果节点是事务的起点，这里将没有父span - 对于这种情况， 使用值-1来表示这个span是事务的根span。

> Google Dapper 和 NAVER Pinpoint在术语上的不同
>
> Pinpoint中的术语"TransactionId"和google dapper中的术语"TraceId"有相同的含义。而Pinpoint中的术语"TraceId"引用到多个key的集合。

## TraceId如何工作

下图描述TraceId的行为，在4个节点之间执行了3次的RPC调用：

![](https://github.com/naver/pinpoint/raw/master/doc/img/td_figure2.png)

图2： TraceId行为示例

在图2中，TransactionId (TxId) 体现了三次不同的RPC作为单个事务被相互关联。但是，TransactionId 本身不能精确描述PRC之间的关系。为了识别PRC之间的关系，需要SpanId 和 ParentSpanId (pSpanId). 假设一个节点是Tomcat，可以将SpanId想象为处理HTTP请求的线程，ParentSpanId代表发起这个RPC调用的SpanId.

使用TransactionId，Pinpoint可以发现关联的n个Span，并使用SpanId和ParentSpanId将这n个span排列为继承树结构。

SpanId 和 ParentSpanId 是 64位长度的整型。可能发生冲突，因为这个数字是任意生成的，但是考虑到值的范围可以从-9223372036854775808到9223372036854775807，不太可能发生冲突. 如果key之间出现冲突，Pinpoint和Google Dapper系统，会让开发人员知道发生了什么，而不是解决冲突。

TransactionId 由 AgentIds, JVM (java虚拟机)启动时间, 和 SequenceNumbers/序列号组成.

- AgentId: 当Jvm启动时用户创建的ID; 必须在pinpoinit安装的全部服务器集群中全局唯一. 最简单的让它保持唯一的方法是使用hostname($HOSTNAME)，因为hostname一般不会重复. 如果需要在服务器集群中运行多个JVM，请在hostname前面增加一个前缀来避免重复。
- JVM 启动时间: 需要用来保证从0开始的SequenceNumber的唯一性. 当用户错误的创建了重复的AgentId时这个值可以用来预防ID冲突。
- SequenceNumber: Pinpoint agent 生成的ID, 从0开始连续自增;为每个消息生成一个.

Dapper 和 [Zipkin](https://github.com/twitter/zipkin), Twitter的一个分布式系统跟踪平台, 生成随机TraceIds (Pinpoint是TransactionIds) 并将冲突情况视为正常。然而, 在Pinpiont中我们想避免冲突的可能，因此实现了上面描述的系统。有两种选择：一是数据量小但是冲突的可能性高，二是数据量大但是冲突的可能性低。我们选择了第二种。

可能有更好的方式来处理transaction。我们起先有一个想法，通过中央key服务器来生成key。如果实现这个模式，可能导致性能问题和网络错误。因此，大量生成key被考虑作为备选。后面这个方法可能被开发。现在采用简单方法。在pinpoint中，TransactionId被当成可变数据来对待。

# 字节码增强，无需代码修改

前面我们解释了分布式事务跟踪。实现的方法之一是开发人员自己修改代码。当发生RPC调用时容许开发人员添加标签信息。但是，修改代码会成为包袱，即使这样的功能对开发人员非常有用。

Twitter的 Zipkin 使用修改过的类库和它自己的容器(Finagle)来提供分布式事务跟踪的功能。但是，它要求在需要时修改代码。我们期望功能可以不修改代码就工作并希望得到代码级别的可见性。为了解决这个问题，pinpoint中使用了字节码增强技术。Pinpoint agent干预发起RPC的代码以此来自动处理标签信息。

## 克服字节码增强的缺点

字节码增强在手工方法和自动方法两者之间属于自动方法。

- 手工方法： 开发人员使用ponpoint提供的API在关键点开发记录数据的代码
- 自动方法： 开发人员不需要代码改动，因为pinpoint决定了哪些API要调节和开发

下面是每个方法的优点和缺点：

Table1 每个方法的优缺点

|        | 优点 | 缺点 |
|--------|--------|--------|
| 手工跟踪 | 1. 要求更少开发资源 2. API可以更简单并最终减少bug的数量 | 1. 开发人员必须修改代码 2. 跟踪级别低|
| 自动跟踪 | 1. 开发人员不需要修改代码 2. 可以收集到更多精确的数据因为有字节码中的更多信息 | 1. 在开发pinpoint时，和实现一个手工方法相比，需要10倍开销来实现一个自动方法 2. 需要更高能力的开发人员，可以立即识别需要跟踪的类库代码并决定跟踪点 3. 增加bug发生的可能性，因为使用了如字节码增强这样的高级开发技巧 |

字节码增强是一种高难度和高风险的技术。但是，综合考虑使用这种技术开所需要的资源和难度，使用它仍然有很多的益处。


虽然它需要大量的开发资源，在开发服务上它需要很少的资源。例如，下面展示了使用字节码增强的自动方法和使用类库的手工方法(在这里的上下文中，开销是为澄清而假设的随机数)之间的开销。

- 自动方法: 总共 100

    - Pinpoint开发开销: 100
    - 服务实施的开销: 0

- 手工方法: 总共 30

    - Pinpoint开发开销: 20
    - 服务实施的开销: 10

上面的数据告诉我们手工方法比自动方法有更合算。但是，不适用于我们的在NAVER的环境。在NAVER我们有几千个服务，因此在上面的数据中需要修改用于服务实施的开销。如果我们有10个服务需要修改，总开销计算如下：

Pinpoint开发开销 20 + 服务实施开销 10 x 10 = 120

基于这个结果，自动方法是一个更合算的方式。

我们很幸运的在pinpoint团队中拥有很多高能力而专注于Java的开发人员。因此，我们相信克服pinpoint开发中的技术难题只是个时间问题。

## 字节码增强的价值

我们选择字节码增强的理由，除了前面描述的那些外，还有下面的强有力的观点：

### 隐藏API

一旦API被暴露给开发人员使用，我们作为API的提供者，就不能随意的修改API。这样的限制会给我们增加压力。

我们可能修改API来纠正错误设计或者添加新的功能。但是，如果做这些受到限制，对我们来说很难改进API。解决这个问题的最好的答案是一个可升级的系统设计，而每个人都知道这不是一个容易的选择。如果我们不能掌控未来，就不可能创建完美的API设计。

而使用字节码增强技术，我们就不必担心暴露跟踪API而可以持续改进设计，不用考虑依赖关系。对于那些计划使用pinpoint开发应用的人，换一句话说，这代表对于pinpoint开发人员，API是可变的。现在，我们将保留隐藏API的想法，因为改进性能和设计是我们的第一优先级。

## 容易启用或者禁用

使用字节码增强的缺点是当Pinpoint自身类库的采样代码出现问题时可能影响应用。不过，可以通过启用或者禁用pinpoint来解决问题，很简单，因为不需要修改代码。

通过增加下面三行到JVM启动脚本中就可以轻易的为应用启用pinpoint：

    -javaagent:$AGENT_PATH/pinpoint-bootstrap-$VERSION.jar
    -Dpinpoint.agentId=<Agent's UniqueId>
    -Dpinpoint.applicationName=<The name indicating a same service (AgentId collection)>

如果因为pinpoint发生问题，只需要在JVM启动脚本中删除这些配置数据。

## 字节码如何工作

由于字节码增强技术处理java字节码， 有增加开发风险的趋势，同时会降低效率。另外，开发人员更容易犯错。在pinpoint，我们通过抽象出拦截器(interceptor)来改进效率和可达性(accessibility)。pinpoint在类装载时通过介入应用代码为分布式事务和性能信息注入必要的跟踪代码。这会提升性能，因为代码注入是在应用代码中直接实施的。

![](https://github.com/naver/pinpoint/raw/master/doc/img/td_figure3.png)

图3： 字节码增强行为

在pinpoint中，拦截器API在性能数据被记录的地方分开(separated)。为了跟踪，我们添加拦截器到目标方法使得before()方法和after()方法被调用，并在before()方法和after()方法中实现了部分性能数据的记录。使用字节码增强，pinpoint agent可以记录需要方法的数据，只有这样采样数据的大小才能变小。

# pinpoint agent的性能优化

最后，我们描述用于pinpoint agent的性能优化的方式。

## 使用二进制格式(thrift)

通过使用二进制格式([thrift](https://thrift.apache.org/))可以提高编码速度，虽然它使用和调试要难一些。也有利于减少网络使用，因为生成的数据比较小。

## 使用变长编码和格式优化数据记录

如果将一个长整型转换为固定长度的字符串， 数据大小一般是8个字节。然而，如果你用变长编码，数据大小可以是从1到10个字符，取决于给定数字的大小。为了减小数据大小，pinpoint使用thrift的CompactProtocol协议（压缩协议)来编码数据，因为变长字符串和记录数据可以为编码格式做优化。pinpoint agent通过基于跟踪的根方法的时间开始来转换其他的时间来减少数据大小。

图4 说明了上面章节描述的想法：

![](https://github.com/naver/pinpoint/raw/master/doc/img/td_figure4.png)

图4： 固定长度编码和可变长度编码的对比

为了得到关于三个不同方法（见图4）被调用时间的数据，不得不在6个不同的点上测量时间，用固定长度编码这需要48个字节(6 * 8)。

以此同时，pinpoint agent 使用可变长度编码并根据对应的格式记录数据。然后在其他时间点通过和参考点比较来计算时间值（在vector中），根方法的起点被确认为参考点。这只需要占用少量的字节，因为vector使用小数字。图4中消耗了13个字节。

如果执行方法花费了更多时间，即使使用可变长度编码也会增加字节数量。但是，依然比固定长度编码更有效率。

## 用常量表替换重复的API信息，SQL语句和字符串

我们希望pinpoint能开启代码级别的跟踪。然而，存在增大数据大小的问题。每次高精度的数据被发送到服务器将增大数据大小，导致增加网络使用。

为了解决这个问题，我们使用了在远程HBase中创建常量表的策略。例如，每次调用"Method A"的信息被发送到pinpoint collector， 数据大小将很大。pinpoint agent 用一个ID替换"method A"，在HBase中作为一个常量表保存ID和"method A"的信息，然后用ID生成跟踪数据。然后当用户在网站上获取跟踪数据时，pinpoint web在常量表中搜索对应ID的方法信息并组合他们。使用同样的方式来减少SQL或者频繁使用的字符串的数据大小。

## 处理大量请求的采样

我们在线门户服务有海量请求。单个服务每天处理超过200亿请求。容易跟踪这样的请求：方法是添加足够多的网络设施和服务器来跟踪请求并扩展服务器来收集数据。然后，这不是处理这种场景的合算的方法，仅仅是浪费金钱和资源。

在Pinpoint，可以收集采样资料而不必跟踪每个请求。在开发环境中请求量很小，每个数据都收集。而在产品环境请求量巨大，收集小比率的数据如1~5%，足够检查整个应用的状态。有采样后，可以最小化应用的网络开销并降低诸如网络和服务器的设施费用。

> pinpoint采样方法
>
> Pinpoint 支持计数采样，如果设置为10则只采样10分之一的请求。我们计划增加新的采样器来更有效率的收集数据。

注：对应的配置项在agent下的pinpoint.config文件中，默认"profiler.sampling.rate=1"表示全部

## 使用异步数据传输来最小化应用线程中止

pinpoint不阻塞应用线程，因为编码后的数据或者远程消息被其他线程异步传输。

### 使用UDP传输数据

和gogole dapper不同，pinpoint通过网络传输数据来确保数据速度。作为一个服务间使用的通用设施，当数据通讯持续突发时网络会成为问题。在这种情况下，pinpoint agent使用UDP协议来给服务让出网络连接优先级。

> 注意
>
> 数据传输API可以被替换，因为它是接口分离的。可以修改实现为用其他方式存储数据，比如本地文件。

# pinpoint应用示例

这里给出一个例子关于如何从应用获取数据，这样就可以全面的理解前面讲述的内容。

图5 展示了当在 TomcatA 和 TomcatB 中安装pinpoint的数据。可以把单个节点的跟踪数据看成single traction，提现分布式事务跟踪的流程。

![](https://github.com/naver/pinpoint/raw/master/doc/img/td_figure5.png)

图5：示例1：pinpoint应用

下面阐述pinpoint为每个方法做了什么：

1. 当请求到达TomcatA时, Pinpoint agent 产生一个 TraceId.

    - TX_ID: TomcatA\^TIME\^1
    - SpanId: 10
    - ParentSpanId: -1(Root)

2. 从spring MVC 控制器中记录数据
3. 插入HttpClient.execute()方法的调用并在HTTPGet中配置TraceId

    - 创建一个子TraceId

        - TX_ID: TomcatA\^TIME\^1 -> TomcatA\^TIME\^1
        - SPAN_ID: 10 -> 20
        - PARENT_SPAN_ID: -1 -> 10 (父 SpanId)

	- 在HTTP header中配置子 TraceId

        - HttpGet.setHeader(PINPOINT_TX_ID, "TomcatA\^TIME\^1")
        - HttpGet.setHeader(PINPOINT_SPAN_ID, "20")
        - HttpGet.setHeader(PINPOINT_PARENT_SPAN_ID, "10")

4. 传输打好tag的请求到TomcatB.

	- TomcatB 检查传输过来的请求的header

		HttpServletRequest.getHeader(PINPOINT_TX_ID)

	- TomcatB 作为子节点工作因为它识别了header中的TraceId

        - TX_ID: TomcatA\^TIME\^1
        - SPAN_ID: 20
        - PARENT_SPAN_ID: 10

5. 从spring mvc控制器中记录数据并完成请求

    ![](https://github.com/naver/pinpoint/raw/master/doc/img/td_figure6.png)

    图6 示例2：pinpoint应用

6. 当从tomcatB回来的请求完成时，pinpoint agent发送跟踪数据到pinpoint collector就此存储在HBase中
7. 在对tomcatB的HTTP调用结束后，TomcatA的请求也完成了。pinpoint agent发送跟踪数据到pinpoint collector就此存储在HBase中
8. UI从HBase中读取跟踪数据并通过排序树来创建调用栈

# 结论

pinpoint是和应用一起运行的另外的应用。使用字节码增强使得pinpoint看上去不需要代码修改。通常，字节码增强技术让应用容易造成风险。如果问题发生在pinpoint中，它会影响应用。目前，我们专注于改进pinpoint的性能和设计，而不是移除这样的威胁，因为我们任务这些让pinpoint更加有价值。因此你需要决定是否使用pinpoint。

我们还是有大量的工作需要完成来改进pinpoint，尽管不完整，pinpoint还是作为开源项目发布了。我们将持续努力开发并改进pinpoint以便满足你的期望。

> Woonduk Kang编写
>
> 在2011年, 关于我自己我这样写到 — 作为一个开发人员，我想开发人们愿意付款的软件程序，像Microsoft 或者 Oracle. 当Pinpoint被作为一个开源项目启动，看上去我的梦想稍微实现了一点。目前， 我的愿望是让pinpoint对用户更加有价值和更惹人喜欢.
