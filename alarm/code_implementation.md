现有代码实现
============

# 代码入口

### applicationContext-web.xml

文件路径: pinpoint/web/src/main/resources/applicationContext-web.xml

导入的配置文件有hbase.properties和jdbc.properties：

```java
<bean id="propertyConfigurer" class="org.springframework.beans.factory.config.PropertyPlaceholderConfigurer">
    <property name="locations">
        <list>
            <value>classpath:hbase.properties</value>
            <value>classpath:jdbc.properties</value>
        </list>
    </property>
</bean>
```

其他导入的spring配置文件：

```java
<import resource="classpath:applicationContext-hbase.xml" />
<import resource="classpath:applicationContext-datasource.xml" />
<import resource="classpath:applicationContext-dao-config.xml" />
<import resource="classpath:applicationContext-cache.xml" />
<import resource="classpath:applicationContext-websocket.xml" />
```



# 批处理

### 类 BatchConfiguration

文件路径：pinpoint/web/src/main/java/com/navercorp/pinpoint/web/batch/BatchConfiguration.java

```java
@Configuration
@Conditional(BatchConfiguration.Condition.class)
@ImportResource("classpath:/batch/applicationContext-batch-schedule.xml")
public class BatchConfiguration{
    static class Condition implements ConfigurationCondition {
        @Override
        public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
            ......
            Resource resource = context.getResourceLoader().getResource("classpath:/batch.properties")
            ......
            final String enable = properties.getProperty("batch.enable");
            ......
        }
    }
}
```

Condition中会读取配置文件batch.properties中的配置项batch.enable，默认是false。因此如果要开启批处理功能，必须设置batch.enable=true。

### applicationContext-batch-schedule.xml

文件路径为：pinpoint/web/src/main/resources/batch/applicationContext-batch-schedule.xml

```java
<task:scheduled-tasks scheduler="scheduler">
    <task:scheduled ref="batchJobLauncher" method="alarmJob" cron="0 0/3 * * * *" />
</task:scheduled-tasks>
```

为了测试方便，可以修改cron表达式为 cron="*/5 * * * *"，每5秒钟执行一次。

### batch.properties

文件路径为：pinpoint/web/src/main/resources/batch.properties

```xml
#batch enable config
batch.enable=true

#batch server ip to execute batch
batch.server.ip=127.0.0.1
```

设置batch.enable=true，另外设置batch.server.ip=127.0.0.1这样每台pinpoint web都会跑批处理。如果安装有多台pinpoint web，可以设置为其中一台的IP。

# 数据源

### applicationContext-datasource.xml

文件路径：pinpoint/web/src/main/resources/applicationContext-datasource.xml

```xml
<!-- DataSource Configuration -->
<bean id="dataSource" class="org.apache.commons.dbcp2.BasicDataSource" destroy-method="close">
    <property name="driverClassName" value="${jdbc.driverClassName}"/>
    <property name="url" value="${jdbc.url}"/>
    <property name="username" value="${jdbc.username}"/>
    <property name="password" value="${jdbc.password}"/>
    ......
</bean>
```

定义了名为dataSource的数据源给其他spring bean使用，配置信息来自jdbc.properties。

### jdbc.properties

文件路径：pinpoint/web/src/main/resources/jdbc.properties

```bash
jdbc.driverClassName=com.mysql.jdbc.Driver
jdbc.url=jdbc:mysql://localhost:13306/pinpoint?characterEncoding=UTF-8
jdbc.username=admin
jdbc.password=admin
```

定义了名为dataSource的数据源，使用mysql。

