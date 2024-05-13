RocketMQ源码调试

### 下载源码并导入到IDEA中

从github上面下载RocketMQ的源码到本地，[下载地址](https://github.com/apache/rocketmq),本源码采用的是rocketmq-all-4.9.2，然后导入到idea开发工具中

导入后的源码如下图

![3464a2b4a2509b6023fb1b4baf44d47](.\pic\3464a2b4a2509b6023fb1b4baf44d47.png)

### 调试RocketMQ源码

#### 1、修改配置文件信息

在RocketMQ运行主目录创建home文件夹，并在文件夹中创建相应的conf、logs、store文件夹，然后从RocketMQ distribution部署目录中将broker.conf、
logback_broker.xml、logback_namesrv.xml等文件复制到conf目录中。如图

![1644371169](.\pic\1644371169.png)

修改braoker.conf文件如下

```properties
brokerClusterName = DefaultCluster
brokerName = broker-a
brokerId = 0
brokerIP1=127.0.0.1
#nameServer地址,分号分割
namesrvAddr=127.0.0.1:9876
deleteWhen = 04
fileReservedTime = 48
brokerRole = ASYNC_MASTER
flushDiskType = ASYNC_FLUSH
#diskSpaceWarningLevelRatio=99

#brokerIP1=127.0.0.1

#存储路径
storePathRootDir=C:\\work\\IDEAWorkSpace\\rookie-project\\source_github\\apache\\rocketmq\\home\\store
#CommitLog存储路径
storePathCommitLog=C:\\work\\IDEAWorkSpace\\rookie-project\\source_github\\apache\\rocketmq\\home\\store\\commitlog
#消费队列存储路径
storePathConsumeQueue=C:\\work\\IDEAWorkSpace\\rookie-project\\source_github\\apache\\rocketmq\\home\\consumequeue
#消息索引存储路径
storePathIndex=C:\\work\\IDEAWorkSpace\\rookie-project\\source_github\\apache\\rocketmq\\home\\store\\index
#checkpoint文件存储路径
storeCheckpoint=C:\\work\\IDEAWorkSpace\\rookie-project\\source_github\\apache\\rocketmq\\home\\store\\checkpoint
#abort文件存储路径
abortFile=C:\\work\\IDEAWorkSpace\\rookie-project\\source_github\\apache\\rocketmq\\home\\store\\abort
```

修改logback_broker.xml、logback_namesrv.xml文件

在logback_broker.xml文件中添加

```xml
<property name="log.path" value="./home/logs"/>
```

并将所有的user.home替换为log.path，同理修改logback_namesrv.xml文件，方便查找程序运行的日志

修改后的logback_broker.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!--
  Licensed to the Apache Software Foundation (ASF) under one or more
  contributor license agreements.  See the NOTICE file distributed with
  this work for additional information regarding copyright ownership.
  The ASF licenses this file to You under the Apache License, Version 2.0
  (the "License"); you may not use this file except in compliance with
  the License.  You may obtain a copy of the License at

      http://www.apache.org/licenses/LICENSE-2.0

  Unless required by applicable law or agreed to in writing, software
  distributed under the License is distributed on an "AS IS" BASIS,
  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  See the License for the specific language governing permissions and
  limitations under the License.
  -->

<configuration>


    <property name="log.path" value="./home/logs"/>


    <appender name="DefaultAppender"
              class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${log.path}/logs/rocketmqlogs/broker_default.log</file>
        <append>true</append>
        <rollingPolicy class="ch.qos.logback.core.rolling.FixedWindowRollingPolicy">
            <fileNamePattern>${log.path}/logs/rocketmqlogs/otherdays/broker_default.%i.log.gz</fileNamePattern>
            <minIndex>1</minIndex>
            <maxIndex>10</maxIndex>
        </rollingPolicy>
        <triggeringPolicy class="ch.qos.logback.core.rolling.SizeBasedTriggeringPolicy">
            <maxFileSize>100MB</maxFileSize>
        </triggeringPolicy>
        <encoder>
            <pattern>%d{yyy-MM-dd HH:mm:ss,GMT+8} %p %t - %m%n</pattern>
            <charset class="java.nio.charset.Charset">UTF-8</charset>
        </encoder>
    </appender>

    <appender name="RocketmqBrokerAppender_inner"
              class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${log.path}/logs/rocketmqlogs/broker.log</file>
        <append>true</append>
        <rollingPolicy class="ch.qos.logback.core.rolling.FixedWindowRollingPolicy">
            <fileNamePattern>${log.path}/logs/rocketmqlogs/otherdays/broker.%i.log.gz</fileNamePattern>
            <minIndex>1</minIndex>
            <maxIndex>20</maxIndex>
        </rollingPolicy>
        <triggeringPolicy class="ch.qos.logback.core.rolling.SizeBasedTriggeringPolicy">
            <maxFileSize>128MB</maxFileSize>
        </triggeringPolicy>
        <encoder>
            <pattern>%d{yyy-MM-dd HH:mm:ss,GMT+8} %p %t - %m%n</pattern>
            <charset class="java.nio.charset.Charset">UTF-8</charset>
        </encoder>
    </appender>
    <appender name="RocketmqBrokerAppender" class="ch.qos.logback.classic.AsyncAppender">
        <appender-ref ref="RocketmqBrokerAppender_inner"/>
    </appender>

    <appender name="RocketmqProtectionAppender_inner"
              class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${log.path}/logs/rocketmqlogs/protection.log</file>
        <append>true</append>
        <rollingPolicy class="ch.qos.logback.core.rolling.FixedWindowRollingPolicy">
            <fileNamePattern>${log.path}/logs/rocketmqlogs/otherdays/protection.%i.log.gz</fileNamePattern>
            <minIndex>1</minIndex>
            <maxIndex>10</maxIndex>
        </rollingPolicy>
        <triggeringPolicy class="ch.qos.logback.core.rolling.SizeBasedTriggeringPolicy">
            <maxFileSize>100MB</maxFileSize>
        </triggeringPolicy>
        <encoder>
            <pattern>%d{yyy-MM-dd HH:mm:ss,GMT+8} - %m%n</pattern>
            <charset class="java.nio.charset.Charset">UTF-8</charset>
        </encoder>
    </appender>
    <appender name="RocketmqProtectionAppender" class="ch.qos.logback.classic.AsyncAppender">
        <appender-ref ref="RocketmqProtectionAppender_inner"/>
    </appender>

    <appender name="RocketmqWaterMarkAppender_inner"
              class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${log.path}/logs/rocketmqlogs/watermark.log</file>
        <append>true</append>
        <rollingPolicy class="ch.qos.logback.core.rolling.FixedWindowRollingPolicy">
            <fileNamePattern>${log.path}/logs/rocketmqlogs/otherdays/watermark.%i.log.gz</fileNamePattern>
            <minIndex>1</minIndex>
            <maxIndex>10</maxIndex>
        </rollingPolicy>
        <triggeringPolicy class="ch.qos.logback.core.rolling.SizeBasedTriggeringPolicy">
            <maxFileSize>100MB</maxFileSize>
        </triggeringPolicy>
        <encoder>
            <pattern>%d{yyy-MM-dd HH:mm:ss,GMT+8} - %m%n</pattern>
            <charset class="java.nio.charset.Charset">UTF-8</charset>
        </encoder>
    </appender>
    <appender name="RocketmqWaterMarkAppender" class="ch.qos.logback.classic.AsyncAppender">
        <appender-ref ref="RocketmqWaterMarkAppender_inner"/>
    </appender>

    <appender name="RocketmqStoreAppender_inner"
              class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${log.path}/logs/rocketmqlogs/store.log</file>
        <append>true</append>
        <rollingPolicy class="ch.qos.logback.core.rolling.FixedWindowRollingPolicy">
            <fileNamePattern>${log.path}/logs/rocketmqlogs/otherdays/store.%i.log.gz</fileNamePattern>
            <minIndex>1</minIndex>
            <maxIndex>10</maxIndex>
        </rollingPolicy>
        <triggeringPolicy class="ch.qos.logback.core.rolling.SizeBasedTriggeringPolicy">
            <maxFileSize>128MB</maxFileSize>
        </triggeringPolicy>
        <encoder>
            <pattern>%d{yyy-MM-dd HH:mm:ss,GMT+8} %p %t - %m%n</pattern>
            <charset class="java.nio.charset.Charset">UTF-8</charset>
        </encoder>
    </appender>
    <appender name="RocketmqStoreAppender" class="ch.qos.logback.classic.AsyncAppender">
        <appender-ref ref="RocketmqStoreAppender_inner"/>
    </appender>

    <appender name="RocketmqRemotingAppender_inner"
              class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${log.path}/logs/rocketmqlogs/remoting.log</file>
        <append>true</append>
        <rollingPolicy class="ch.qos.logback.core.rolling.FixedWindowRollingPolicy">
            <fileNamePattern>${log.path}/logs/rocketmqlogs/otherdays/remoting.%i.log.gz</fileNamePattern>
            <minIndex>1</minIndex>
            <maxIndex>10</maxIndex>
        </rollingPolicy>
        <triggeringPolicy class="ch.qos.logback.core.rolling.SizeBasedTriggeringPolicy">
            <maxFileSize>100MB</maxFileSize>
        </triggeringPolicy>
        <encoder>
            <pattern>%d{yyy-MM-dd HH:mm:ss,GMT+8} %p %t - %m%n</pattern>
            <charset class="java.nio.charset.Charset">UTF-8</charset>
        </encoder>
    </appender>
    <appender name="RocketmqRemotingAppender" class="ch.qos.logback.classic.AsyncAppender">
        <appender-ref ref="RocketmqRemotingAppender_inner"/>
    </appender>

    <appender name="RocketmqStoreErrorAppender_inner"
              class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${log.path}/logs/rocketmqlogs/storeerror.log</file>
        <append>true</append>
        <rollingPolicy class="ch.qos.logback.core.rolling.FixedWindowRollingPolicy">
            <fileNamePattern>${log.path}/logs/rocketmqlogs/otherdays/storeerror.%i.log.gz</fileNamePattern>
            <minIndex>1</minIndex>
            <maxIndex>10</maxIndex>
        </rollingPolicy>
        <triggeringPolicy class="ch.qos.logback.core.rolling.SizeBasedTriggeringPolicy">
            <maxFileSize>100MB</maxFileSize>
        </triggeringPolicy>
        <encoder>
            <pattern>%d{yyy-MM-dd HH:mm:ss,GMT+8} %p %t - %m%n</pattern>
            <charset class="java.nio.charset.Charset">UTF-8</charset>
        </encoder>
    </appender>
    <appender name="RocketmqStoreErrorAppender" class="ch.qos.logback.classic.AsyncAppender">
        <appender-ref ref="RocketmqStoreErrorAppender_inner"/>
    </appender>


    <appender name="RocketmqTransactionAppender_inner"
              class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${log.path}/logs/rocketmqlogs/transaction.log</file>
        <append>true</append>
        <rollingPolicy class="ch.qos.logback.core.rolling.FixedWindowRollingPolicy">
            <fileNamePattern>${log.path}/logs/rocketmqlogs/otherdays/transaction.%i.log.gz</fileNamePattern>
            <minIndex>1</minIndex>
            <maxIndex>10</maxIndex>
        </rollingPolicy>
        <triggeringPolicy class="ch.qos.logback.core.rolling.SizeBasedTriggeringPolicy">
            <maxFileSize>100MB</maxFileSize>
        </triggeringPolicy>
        <encoder>
            <pattern>%d{yyy-MM-dd HH:mm:ss,GMT+8} %p %t - %m%n</pattern>
            <charset class="java.nio.charset.Charset">UTF-8</charset>
        </encoder>
    </appender>
    <appender name="RocketmqTransactionAppender" class="ch.qos.logback.classic.AsyncAppender">
        <appender-ref ref="RocketmqTransactionAppender_inner"/>
    </appender>

    <appender name="RocketmqRebalanceLockAppender_inner"
              class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${log.path}/logs/rocketmqlogs/lock.log</file>
        <append>true</append>
        <rollingPolicy class="ch.qos.logback.core.rolling.FixedWindowRollingPolicy">
            <fileNamePattern>${log.path}/logs/rocketmqlogs/otherdays/lock.%i.log.gz</fileNamePattern>
            <minIndex>1</minIndex>
            <maxIndex>5</maxIndex>
        </rollingPolicy>
        <triggeringPolicy class="ch.qos.logback.core.rolling.SizeBasedTriggeringPolicy">
            <maxFileSize>100MB</maxFileSize>
        </triggeringPolicy>
        <encoder>
            <pattern>%d{yyy-MM-dd HH:mm:ss,GMT+8} %p %t - %m%n</pattern>
            <charset class="java.nio.charset.Charset">UTF-8</charset>
        </encoder>
    </appender>
    <appender name="RocketmqRebalanceLockAppender" class="ch.qos.logback.classic.AsyncAppender">
        <appender-ref ref="RocketmqRebalanceLockAppender_inner"/>
    </appender>

    <appender name="RocketmqFilterAppender_inner"
              class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${log.path}/logs/rocketmqlogs/filter.log</file>
        <append>true</append>
        <rollingPolicy class="ch.qos.logback.core.rolling.FixedWindowRollingPolicy">
            <fileNamePattern>${log.path}/logs/rocketmqlogs/otherdays/filter.%i.log.gz</fileNamePattern>
            <minIndex>1</minIndex>
            <maxIndex>10</maxIndex>
        </rollingPolicy>
        <triggeringPolicy class="ch.qos.logback.core.rolling.SizeBasedTriggeringPolicy">
            <maxFileSize>100MB</maxFileSize>
        </triggeringPolicy>
        <encoder>
            <pattern>%d{yyy-MM-dd HH:mm:ss,GMT+8} %p %t - %m%n</pattern>
            <charset class="java.nio.charset.Charset">UTF-8</charset>
        </encoder>
    </appender>
    <appender name="RocketmqFilterAppender" class="ch.qos.logback.classic.AsyncAppender">
        <appender-ref ref="RocketmqFilterAppender_inner"/>
    </appender>

    <appender name="RocketmqStatsAppender"
              class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${log.path}/logs/rocketmqlogs/stats.log</file>
        <append>true</append>
        <rollingPolicy class="ch.qos.logback.core.rolling.FixedWindowRollingPolicy">
            <fileNamePattern>${log.path}/logs/rocketmqlogs/otherdays/stats.%i.log.gz</fileNamePattern>
            <minIndex>1</minIndex>
            <maxIndex>5</maxIndex>
        </rollingPolicy>
        <triggeringPolicy class="ch.qos.logback.core.rolling.SizeBasedTriggeringPolicy">
            <maxFileSize>100MB</maxFileSize>
        </triggeringPolicy>
        <encoder>
            <pattern>%d{yyy-MM-dd HH:mm:ss,GMT+8} %p - %m%n</pattern>
            <charset class="java.nio.charset.Charset">UTF-8</charset>
        </encoder>
    </appender>

    <appender name="RocketmqCommercialAppender"
              class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${log.path}/logs/rocketmqlogs/commercial.log</file>
        <append>true</append>
        <rollingPolicy class="ch.qos.logback.core.rolling.FixedWindowRollingPolicy">
            <fileNamePattern>${log.path}/logs/rocketmqlogs/otherdays/commercial.%i.log.gz</fileNamePattern>
            <minIndex>1</minIndex>
            <maxIndex>10</maxIndex>
        </rollingPolicy>
        <triggeringPolicy class="ch.qos.logback.core.rolling.SizeBasedTriggeringPolicy">
            <maxFileSize>500MB</maxFileSize>
        </triggeringPolicy>
    </appender>

    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <append>true</append>
        <encoder>
            <pattern>%d{yyy-MM-dd HH\:mm\:ss,GMT+8} %p %t - %m%n</pattern>
            <charset class="java.nio.charset.Charset">UTF-8</charset>
        </encoder>
    </appender>

    <logger name="RocketmqBroker" additivity="false">
        <level value="INFO"/>
        <appender-ref ref="RocketmqBrokerAppender"/>
    </logger>

    <logger name="RocketmqProtection" additivity="false">
        <level value="INFO"/>
        <appender-ref ref="RocketmqProtectionAppender"/>
    </logger>

    <logger name="RocketmqWaterMark" additivity="false">
        <level value="INFO"/>
        <appender-ref ref="RocketmqWaterMarkAppender"/>
    </logger>

    <logger name="RocketmqCommon" additivity="false">
        <level value="INFO"/>
        <appender-ref ref="RocketmqBrokerAppender"/>
    </logger>

    <logger name="RocketmqStore" additivity="false">
        <level value="INFO"/>
        <appender-ref ref="RocketmqStoreAppender"/>
    </logger>

    <logger name="RocketmqStoreError" additivity="false">
        <level value="INFO"/>
        <appender-ref ref="RocketmqStoreErrorAppender"/>
    </logger>

    <logger name="RocketmqTransaction" additivity="false">
        <level value="INFO"/>
        <appender-ref ref="RocketmqTransactionAppender"/>
    </logger>

    <logger name="RocketmqRebalanceLock" additivity="false">
        <level value="INFO"/>
        <appender-ref ref="RocketmqRebalanceLockAppender"/>
    </logger>

    <logger name="RocketmqRemoting" additivity="false">
        <level value="INFO"/>
        <appender-ref ref="RocketmqRemotingAppender"/>
    </logger>

    <logger name="RocketmqStats" additivity="false">
        <level value="INFO"/>
        <appender-ref ref="RocketmqStatsAppender"/>
    </logger>

    <logger name="RocketmqCommercial" additivity="false">
        <level value="INFO"/>
        <appender-ref ref="RocketmqCommercialAppender"/>
    </logger>

    <logger name="RocketmqFilter" additivity="false">
        <level value="INFO"/>
        <appender-ref ref="RocketmqFilterAppender"/>
    </logger>

    <logger name="RocketmqConsole" additivity="false">
        <level value="INFO"/>
        <appender-ref ref="STDOUT"/>
    </logger>

    <root>
        <level value="INFO"/>
        <appender-ref ref="DefaultAppender"/>
    </root>
</configuration>

```

#### 2、启动NameServer

找到namesrv模块，并打开NamesrvStartup.java类，右键main方法左侧的运行按钮，选中Modify Run Configuration...进行信息配置，在Environment variables中输入ROCKETMQ_HOME=C:\work\IDEAWorkSpace\rookie-project\source_github\apache\rocketmq\home即主目录的home，点击Apply即可。配置如图

![1644370779](.\pic\1644370779.png)

点击并运行NamesrvStartup,控制台打印出The Name Server boot success. serializeType=JSON即表示运行成功

#### 3、启动Broker

通配置NameServer类似，打开Broker模块，然后在Environment variables中输入ROCKETMQ_HOME=C:\work\IDEAWorkSpace\rookie-project\source_github\apache\rocketmq\home即主目录的home,同时在Program arguments填写-c C:\work\IDEAWorkSpace\rookie-project\source_github\apache\rocketmq\home\conf\broker.conf信息，点击Apply即可。如图

![1644373759](.\pic\1644373759.png)

点击并运行BrokerStartup，然后打开broker.log文件，未报错则表示Broker启动成功

4、进行消息的发送与消费

分别在example模块中的org.apache.rocketmq.example.quickstart包中的Producer和Consumer类中添加

```java
producer.setNamesrvAddr("127.0.0.1:9876");
```

```java
consumer.setNamesrvAddr("127.0.0.1:9876");
```

如图

![1644374866](.\pic\1644374866.png)

![1644374881](.\pic\1644374881.png)

运行生产者和消费者，可以看到可以正常进行生产消息和消费消息了，说明RocketMQ调试环境已搭建成功









