Alarm
======

> 注：内容翻译自 [官方文档Alarm](https://github.com/naver/pinpoint/blob/master/doc/alarm.md)

Pinpoint-web周期性的检查应用的状态，如果特定前置条件(规则)满足时则触发告警。

这些条件(默认)每3分钟被web模块中的后台批处理程序检查一次，使用最后5分钟的数据。一旦条件满足，批处理程序发送短信/邮件给注册到用户组的用户。

# 用户指南

1. 配置菜单

	![](https://github.com/naver/pinpoint/raw/master/doc/img/alarm/alarm_figure01.gif)

2. 注册用户

	![](https://github.com/naver/pinpoint/raw/master/doc/img/alarm/alarm_figure02.gif)

3. 创建用户组

	![](https://github.com/naver/pinpoint/raw/master/doc/img/alarm/alarm_figure03.gif)

4. 添加用户到用户组

	![](https://github.com/naver/pinpoint/raw/master/doc/img/alarm/alarm_figure04.gif)

5. 设置告警规则

	![](https://github.com/naver/pinpoint/raw/master/doc/img/alarm/alarm_figure05.gif)

## 告警规则

- SLOW COUNT / 慢请求数

	当应用发出的慢请求数量超过配置阈值时触发。

- SLOW RATE / 慢请求比例

	当应用发出的慢请求百分比超过配置阈值时触发。

- ERROR COUNT / 请求失败数

   当应用发出的失败请求数量超过配置阈值时触发。

- ERROR RATE / 请求失败率

	当应用发出的失败请求百分比超过配置阈值时触发。

- TOTAL COUNT / 总数量

	当应用发出的所有请求数量超过配置阈值时触发。
	> 以上规则中，请求是当前应用发送出去的，当前应用是请求的发起者。
	> 以下规则中，请求是发送给当前应用的，当前应用是请求的接收者。

- SLOW COUNT TO CALLEE / 被调用的慢请求数量

	当发送给应用的慢请求数量超过配置阈值时触发。

- SLOW RATE TO CALLEE / 被调用的慢请求比例

   当发送给应用的慢请求百分比超过配置阈值时触发。

- ERROR COUNT TO CALLEE / 被调用的请求错误数

   当发送给应用的请求失败数量超过配置阈值时触发。

- ERROR RATE TO CALLEE / 被调用的请求错误率

   当发送给应用的请求失败百分比超过配置阈值时触发。

- TOTAL COUNT TO CALLEE / 被调用的总数量

	当发送给应用的所有请求数量超过配置阈值时触发。
	> 下面两条规则和请求无关，只涉及到应用的状态

- HEAP USAGE RATE / 堆内存使用率

	当应用的堆内存使用率超过配置阈值时触发。

- JVM CPU USAGE RATE / JVM CPU使用率

   当应用的CPU使用率超过配置阈值时触发。

# 实现和配置

为了使用告警功能，必须通过实现 com.navercorp.pinpoint.web.alarm.AlarmMessageSender并注册为spring managed bean 来实现自己的逻辑以便发送短信和邮件。当告警被触发时， AlarmMessageSender#sendEmail, 和 AlarmMessageSender#sendSms 方法将被调用。

## 实现AlarmMessageSender 并注册 Spring bean

```java
public class AlarmMessageSenderImple implements AlarmMessageSender {

    private final Logger logger = LoggerFactory.getLogger(this.getClass());

    @Override
    public void sendSms(AlarmChecker checker, int sequenceCount) {
        List<String> receivers = userGroupService.selectPhoneNumberOfMember(checker.getUserGroupId());

        if (receivers.size() == 0) {
            return;
        }

        for (String message : checker.getSmsMessage()) {
            logger.info("send SMS : {}", message);

            // TODO Implement logic for sending SMS
        }
    }

    @Override
    public void sendEmail(AlarmChecker checker, int sequenceCount) {
        List<String> receivers = userGroupService.selectEmailOfMember(checker.getUserGroupId());

        if (receivers.size() == 0) {
            return;
        }

        for (String message : checker.getEmailMessage()) {
            logger.info("send email : {}", message);

            // TODO Implement logic for sending email
        }
    }
}
```

```xml
<bean id="AlarmMessageSenderImple" class="com.navercorp.pinpoint.web.alarm.AlarmMessageSenderImple"/>
```

> 注：以上代码copy自原文，谨慎请见，开发时请以英文原文为准。

## 配置批处理属性

设置batch.properties文件中的 batch.enable 标记为true：

	batch.enable=true

这里的 batch.server.ip 配置用于当有多台pinpoint web server时防止并发批处理程序. 仅当服务器IP地址和 batch.server.ip 设置的一致时才执行批处理。(设置为 127.0.0.1 将在所有的web服务器上启动批处理)

	batch.server.ip=X.X.X.X

> 注： 这种防止并发的方式有点简陋而原始，存在单点故障的风险，主要缺陷：万一配置的这台容许批处理的web服务器down机，告警功能就失效了。

## 配置mysql

搭建mysql服务器并在jdbc.properties文件中配置连接信息：

    jdbc.driverClassName=com.mysql.jdbc.Driver
    jdbc.url=jdbc:mysql://localhost:13306/pinpoint?characterEncoding=UTF-8
    jdbc.username=admin
    jdbc.password=admin

运行 [CreateTableStatement-mysql.sql](https://github.com/naver/pinpoint/blob/master/web/src/main/resources/sql/CreateTableStatement-mysql.sql) 和 [SpringBatchJobReositorySchema-mysql.sql](https://github.com/naver/pinpoint/blob/master/web/src/main/resources/sql/SpringBatchJobReositorySchema-mysql.sql) 来创建表.

## 其他

1. 可以在独立进程中启动告警批处理

	使用Pinpoint-web模块中的 [applicationContext-alarmJob.xml](https://github.com/naver/pinpoint/blob/master/web/src/main/resources/batch/applicationContext-alarmJob.xml) 文件简单启动spring batch 任务.
2. 通过修改 [applicationContext-batch-schedule.xml](https://github.com/naver/pinpoint/blob/master/web/src/main/resources/batch/applicationContext-batch-schedule.xml) 文件中的cron 表达式来修改批处理周期：

    ```xml
    <task:scheduled-tasks scheduler="scheduler">
        <task:scheduled ref="batchJobLauncher" method="alarmJob" cron="0 0/3 * * * *" />
    </task:scheduled-tasks>
    ```

3. 提高告警批处理性能的方式

	告警批处理被设计为并发运行. 如果有很多应用注册有告警，可以通过修改 [applicationContext-batch.xml](https://github.com/naver/pinpoint/blob/master/web/src/main/resources/batch/applicationContext-batch.xml) 文件中的 pool-size 来增大executor的线程池大小.

    注意增大这个值会导致更高的资源使用。

    ```xml
    <task:executor id="poolTaskExecutorForPartition" pool-size="1" />
    ```

    如果有应用注册有很多告警，可以设置注册在 [applicationContext-batch.xml](https://github.com/naver/pinpoint/blob/master/web/src/main/resources/batch/applicationContext-batch.xml) 文件中的alarmStep来并发运行：

    ```xml
    <step id="alarmStep" xmlns="http://www.springframework.org/schema/batch">
        <tasklet task-executor="poolTaskExecutorForStep" throttle-limit="3">
            <chunk reader="reader" processor="processor" writer="writer" commit-interval="1"/>
        </tasklet>
    </step>
    <task:executor id="poolTaskExecutorForStep" pool-size="10" />
    ```

