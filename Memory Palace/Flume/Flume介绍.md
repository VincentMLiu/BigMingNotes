# Flume概念

Flume是Cloudera贡献给Apache的一个日志采集框架。

Flume是一种**分布式**、**高可靠**、**高可用**的服务，用于高效地**收集、聚合和移动海量日志数据**。它具有基于流数据流的简单灵活的体系结构。它具有健壮性和容错性，具有可调的可靠性机制和许多故障切换和恢复机制。它使用一个简单的可扩展数据模型，允许在线分析应用程序。

![Agent component diagram](https://gitee.com/vincentmliu/big-data-learning-materials/raw/master/trainning_materials/Flume/md_pics/DevGuide_image00.png)

通过source从外部数据源获取数据，source将日志转换成Event，并将Event存储在一个或者多个Channel中。Sink从Channel中获取Event，并将Event发送至下一个flume或者储存至持久化存储中。

官网

https://flume.apache.org/

文档

https://flume.apache.org/releases/content/1.9.0/FlumeUserGuide.html#setting-multi-agent-flow





## 高可靠

通过Transaction来控制事务，source通过put机制将event发送至channel。sink则通过poll机制来获取event，只有向sink的目标发送成功后，才会向channel告知，并弹出event。sink和source通过channel解耦合。

## 高可用

channel一般采用本地文件或高可用存储（比如kafka）来实现，进程失效时可通过check_point方便恢复。

## 分布式

本身不是一个分布式系统。需要借助HDFS，Kafka等源或者目标来实现分布式。

![flume分布式么](https://gitee.com/vincentmliu/big-data-learning-materials/raw/master/trainning_materials/Flume/md_pics/flume_distributed)



Flume本身就像一个农民，专门给数据中台或者数仓来采矿。Flume中文翻译是管道的意思，本身不处理数据，只拆分并采集数据。



## 术语

***Agent*** ：Agent 是一个 JVM 进程，它以事件的形式将数据从源头送至目的。一个Agent 主要有 3 个部分组成，Source、Channel、Sink。

***Source***：Source 是负责接收数据到 Flume Agent 的组件。Source 组件可以处理各种类型、各种格式的日志数据，包括 avro、thrift、exec、jms、spooling directory、netcat、taildir、sequence generator、syslog、http、legacy。

***Sink***：Sink 不断地轮询 Channel 中的事件且批量地移除它们，并将这些事件批量写入到存储
或索引系统、或者被发送到另一个 Flume Agent。
Sink 组件目的地包括 hdfs、ES、logger、avro、thrift、ipc、file、HBase、solr、自定
义。

***Channel***：Channel 是位于 Source 和 Sink 之间的缓冲区。因此，Channel 允许 Source 和 Sink 运作在不同的速率上。Channel 是线程安全的，可以同时处理几个 Source 的写入操作和几个
Sink 的读取操作

**Event**：Event是Flume各组件间最基本数据传输单元，数据在source中被转换成Event，最后传输至sink。

Event 由 Header 和 Body 两部分组成，Header 用来存放该 event 的一些属性，为 K-V 结构，Body 用来存放该条数据，形式为字节数组。

![image-20211217162951097](https://gitee.com/vincentmliu/big-data-learning-materials/raw/master/trainning_materials/Flume/md_pics/Event)



**Selector:** flume通过配置selector来支持将一个source中的数据发送至多个channel。selector默认为replicating，也可以配置为multiplexing。

**Interceptor**： flume可以在Source将Event put给Channel之前，过滤或修改Event。通过配置单个或者链式Interceptor来实现。







# Flume配置

Flume通过配置文件来构建数据流的结构。配置信息可以保存在本地文件和zookeeper中，格式需要符合java properties的文件格式。

## 配置样例

```properties
# example.conf: A single-node Flume configuration

# Name the components on this agent
a1.sources = r1
a1.sinks = k1
a1.channels = c1

# Describe/configure the source
a1.sources.r1.type = netcat
a1.sources.r1.bind = localhost
a1.sources.r1.port = 44444

# Describe the sink
a1.sinks.k1.type = logger

# Use a channel which buffers events in memory
a1.channels.c1.type = memory
a1.channels.c1.capacity = 1000
a1.channels.c1.transactionCapacity = 100

# Bind the source and sink to the channel
a1.sources.r1.channels = c1
a1.sinks.k1.channel = c1
```

* 每个组件都有一个独立的名称；
* 单台机器上多个flume agent时，每个agent的名称不能重复；
* 配置文件中可以包含环境变量，例如，${NC_PORT}

## 样例演示

```shell
./flume-ng agent --conf conf --conf-file ../conf/netcat.conf --name a1 -Dflume.root.logger=INFO,console
```



启动一个flume agent，去接收localhost:44444端口的数据

```shell
telnet localhost 44444
```

测试是否能够接收数据

## 自定义插件包

1.9版本建议将自定义插件包配置在$FLUME_HOME/plugins.d目录中，启动时会自动加载到CLASSPATH中。

**插件包目录规范：**

1. lib目录加载插件包；

2. libext加载插件的依赖包；

3. native加载本地方法包。

   

```shell
plugins.d/
plugins.d/custom-source-1/
plugins.d/custom-source-1/lib/my-source.jar
plugins.d/custom-source-1/libext/spring-core-2.5.6.jar
plugins.d/custom-source-2/
plugins.d/custom-source-2/lib/custom.jar
plugins.d/custom-source-2/native/gettext.so
```



## 构建流

Flume支持各类RPC机制，Avro、Thrift、Syslog、Netcat。所以可以将多个flume agent构建成不同形式的数据流。

### 串联

![Two agents communicating over Avro RPC](https://gitee.com/vincentmliu/big-data-learning-materials/raw/master/trainning_materials/Flume/md_pics/series_connection)

从一个agent发送数据到下一个agent。

### 归并

![A fan-in flow using Avro RPC to consolidate events in one place](https://gitee.com/vincentmliu/big-data-learning-materials/raw/master/trainning_materials/Flume/md_pics/Consolidation)

多个agent汇聚到一个agent中。



### 扇形

![A fan-out flow using a (multiplexing) channel selector](https://gitee.com/vincentmliu/big-data-learning-materials/raw/master/trainning_materials/Flume/md_pics/Multiplexing)



注: 可参考selector

replicating是同时往三个channel发送相同的数据，也就是一份数据复制三份。

multiplexing是根据event header中的信息来决定向哪个channel发送数据。

```properties
# channel selector configuration
agent_foo.sources.avro-AppSrv-source1.selector.type = multiplexing
agent_foo.sources.avro-AppSrv-source1.selector.header = State
agent_foo.sources.avro-AppSrv-source1.selector.mapping.CA = mem-channel-1
agent_foo.sources.avro-AppSrv-source1.selector.mapping.AZ = file-channel-2
agent_foo.sources.avro-AppSrv-source1.selector.mapping.NY = mem-channel-1 file-channel-2
agent_foo.sources.avro-AppSrv-source1.selector.optional.CA = mem-channel-1 file-channel-2
agent_foo.sources.avro-AppSrv-source1.selector.mapping.AZ = file-channel-2
agent_foo.sources.avro-AppSrv-source1.selector.default = mem-channel-1
```

header中key为CA的发送到mem-channel-1，依此类推。

另外channel selector还有一个optional选项，如果mapping对应的channel写入失败会导致transcation失败，并且重试所有的channel。但是配置optional中的channel如果写入失败则会被直接忽略。如果同时配置了mapping和optional，会被认为是mapping。

如果header中不存在相应的key和value，event会被写入default中的channel



##  interceptor

样例：

```properties
a1.sources = r1
a1.sinks = k1
a1.channels = c1
a1.sources.r1.interceptors = i1 i2
a1.sources.r1.interceptors.i1.type = org.apache.flume.interceptor.HostInterceptor$Builder
a1.sources.r1.interceptors.i1.preserveExisting = false
a1.sources.r1.interceptors.i1.hostHeader = hostname
a1.sources.r1.interceptors.i2.type = org.apache.flume.interceptor.TimestampInterceptor$Builder
a1.sinks.k1.filePrefix = FlumeData.%{CollectorHost}.%Y-%m-%d
a1.sinks.k1.channel = c1
```





## Flume1.9版本中开箱即用的组件

![image-20211220172319813](https://gitee.com/vincentmliu/big-data-learning-materials/raw/master/trainning_materials/Flume/md_pics/flume_componnent)



# 💡Flume 自定义插件



官方开发文档

https://flume.apache.org/releases/content/1.9.0/FlumeDeveloperGuide.html

## 自定义sink开发

Sink的主要工作是从Channel获取Event, 并将Event发送到下一个进程或存储中。一个Sink只对应一个Channel。

启动时，Flume框架会生成一个SinkRunner实例。

1. 首先通过configure方法对sink实例进行配置；
2. SinkRunner调用start()方法，实例化sink中的各种状态；
3. **Sink.process()**方法中实现Event处理，并将event发送至下一跳；
4. Sink.stop()方法用于释放资源。

```Java
public class MySink extends AbstractSink implements Configurable {
  private String myProp;

  @Override
  public void configure(Context context) {
    String myProp = context.getString("myProp", "defaultValue");

    // Process the myProp value (e.g. validation)

    // Store myProp for later retrieval by process() method
    this.myProp = myProp;
  }

  @Override
  public void start() {
    // Initialize the connection to the external repository (e.g. HDFS) that
    // this Sink will forward Events to ..
  }

  @Override
  public void stop () {
    // Disconnect from the external respository and do any
    // additional cleanup (e.g. releasing resources or nulling-out
    // field values) ..
  }

  @Override
  public Status process() throws EventDeliveryException {
    Status status = null;

    // Start transaction
    Channel ch = getChannel();
    Transaction txn = ch.getTransaction();
    txn.begin();
    try {
      // This try clause includes whatever Channel operations you want to do

      Event event = ch.take();

      // Send the Event to the external repository.
      // storeSomeData(e);

      txn.commit();
      status = Status.READY;
    } catch (Throwable t) {
      txn.rollback();

      // Log exception, handle individual exceptions as needed

      status = Status.BACKOFF;

      // re-throw all Errors
      if (t instanceof Error) {
        throw (Error)t;
      }
    }
    return status;
  }
}

```

## 自定义Source开发

Source的工作是从外部获取数据，并将数据转换为Event, 最后将数据发送给Channel。

和Sink类似，Flume框架会为每个source启动一个PollableSourceRunner实例。

1. 首先通过configure方法对source实例进行配置；
2. PollableSourceRunner调用start()方法，实例化source中的各种状态；
3. **PollableSourceRunner.process()**方法中实现Event处理，并将event推送至channel；
4. PollableSourceRunner.stop()方法用于释放资源。

```Java
public class MySource extends AbstractSource implements Configurable, PollableSource {
  private String myProp;

  @Override
  public void configure(Context context) {
    String myProp = context.getString("myProp", "defaultValue");

    // Process the myProp value (e.g. validation, convert to another type, ...)

    // Store myProp for later retrieval by process() method
    this.myProp = myProp;
  }

  @Override
  public void start() {
    // Initialize the connection to the external client
  }

  @Override
  public void stop () {
    // Disconnect from external client and do any additional cleanup
    // (e.g. releasing resources or nulling-out field values) ..
  }

  @Override
  public Status process() throws EventDeliveryException {
    Status status = null;

    try {
      // This try clause includes whatever Channel/Event operations you want to do

      // Receive new data
      Event e = getSomeData();

      // Store the Event into this Source's associated Channel(s)
      getChannelProcessor().processEvent(e);

      status = Status.READY;
    } catch (Throwable t) {
      // Log exception, handle individual exceptions as needed

      status = Status.BACKOFF;

      // re-throw all Errors
      if (t instanceof Error) {
        throw (Error)t;
      }
    } finally {
      txn.close();
    }
    return status;
  }
}
```



与Sink略有区别。

除了PollableSource之外，还有另一类source，EventDrivenSource。EventDrivenSource不是通过线程的Runnable实例驱动的，EventDrivenSource自己管理其生命周期。

例如HTTPSource。




# Flume 监控



## JMX

Flume主要支持使用JMX来对Flume程序进行监控。

**监控方法**

在flume-env.sh脚本中添加JMX参数，

`export JAVA_OPTS=”-Dcom.sun.management.jmxremote  -Dcom.sun.management.jmxremote.port=5445  -Dcom.sun.management.jmxremote.authenticate=false  -Dcom.sun.management.jmxremote.ssl=false`”



### 组件和指标对照表

#### Sources 1

|                          | Avro | Exec | HTTP | JMS  | Kafka | MultiportSyslogTCP | Scribe |
| ------------------------ | ---- | ---- | ---- | ---- | ----- | ------------------ | ------ |
| AppendAcceptedCount      | x    |      |      |      |       |                    |        |
| AppendBatchAcceptedCount | x    |      | x    | x    |       |                    |        |
| AppendBatchReceivedCount | x    |      | x    | x    |       |                    |        |
| AppendReceivedCount      | x    |      |      |      |       |                    |        |
| ChannelWriteFail         | x    |      | x    | x    | x     | x                  | x      |
| EventAcceptedCount       | x    | x    | x    | x    | x     | x                  | x      |
| EventReadFail            |      |      | x    | x    | x     | x                  | x      |
| EventReceivedCount       | x    | x    | x    | x    | x     | x                  | x      |
| GenericProcessingFail    |      |      | x    |      |       | x                  |        |
| KafkaCommitTimer         |      |      |      |      | x     |                    |        |
| KafkaEmptyCount          |      |      |      |      | x     |                    |        |
| KafkaEventGetTimer       |      |      |      |      | x     |                    |        |
| OpenConnectionCount      | x    |      |      |      |       |                    |        |

#### Sources 2

|                          | SequenceGenerator | SpoolDirectory | SyslogTcp | SyslogUDP | Taildir | Thrift |
| ------------------------ | ----------------- | -------------- | --------- | --------- | ------- | ------ |
| AppendAcceptedCount      |                   |                |           |           |         | x      |
| AppendBatchAcceptedCount | x                 | x              |           |           | x       | x      |
| AppendBatchReceivedCount |                   | x              |           |           | x       | x      |
| AppendReceivedCount      |                   |                |           |           |         | x      |
| ChannelWriteFail         | x                 | x              | x         | x         | x       | x      |
| EventAcceptedCount       | x                 | x              | x         | x         | x       | x      |
| EventReadFail            |                   | x              | x         | x         | x       |        |
| EventReceivedCount       |                   | x              | x         | x         | x       | x      |
| GenericProcessingFail    |                   | x              |           |           | x       |        |
| KafkaCommitTimer         |                   |                |           |           |         |        |
| KafkaEmptyCount          |                   |                |           |           |         |        |
| KafkaEventGetTimer       |                   |                |           |           |         |        |
| OpenConnectionCount      |                   |                |           |           |         |        |

#### Sinks 1

|                        | Avro/Thrift | AsyncHBase | ElasticSearch | HBase | HBase2 |
| ---------------------- | ----------- | ---------- | ------------- | ----- | ------ |
| BatchCompleteCount     | x           | x          | x             | x     | x      |
| BatchEmptyCount        | x           | x          | x             | x     | x      |
| BatchUnderflowCount    | x           | x          | x             | x     | x      |
| ChannelReadFail        | x           |            |               |       | x      |
| ConnectionClosedCount  | x           | x          | x             | x     | x      |
| ConnectionCreatedCount | x           | x          | x             | x     | x      |
| ConnectionFailedCount  | x           | x          | x             | x     | x      |
| EventDrainAttemptCount | x           | x          | x             | x     | x      |
| EventDrainSuccessCount | x           | x          | x             | x     | x      |
| EventWriteFail         | x           |            |               |       | x      |
| KafkaEventSendTimer    |             |            |               |       |        |
| RollbackCount          |             |            |               |       |        |

#### Sinks 2

|                        | HDFSEvent | Hive | Http | Kafka | Morphline | RollingFile |
| ---------------------- | --------- | ---- | ---- | ----- | --------- | ----------- |
| BatchCompleteCount     | x         | x    |      |       | x         |             |
| BatchEmptyCount        | x         | x    |      | x     | x         |             |
| BatchUnderflowCount    | x         | x    |      | x     | x         |             |
| ChannelReadFail        | x         | x    | x    | x     | x         | x           |
| ConnectionClosedCount  | x         | x    |      |       |           | x           |
| ConnectionCreatedCount | x         | x    |      |       |           | x           |
| ConnectionFailedCount  | x         | x    |      |       |           | x           |
| EventDrainAttemptCount | x         | x    | x    |       | x         | x           |
| EventDrainSuccessCount | x         | x    | x    | x     | x         | x           |
| EventWriteFail         | x         | x    | x    | x     | x         | x           |
| KafkaEventSendTimer    |           |      |      | x     |           |             |
| RollbackCount          |           |      |      | x     |           |             |

#### Channels

|                                 | File | Kafka | Memory | PseudoTxnMemory | SpillableMemory |
| ------------------------------- | ---- | ----- | ------ | --------------- | --------------- |
| ChannelCapacity                 | x    |       | x      |                 | x               |
| ChannelSize                     | x    |       | x      | x               | x               |
| CheckpointBackupWriteErrorCount | x    |       |        |                 |                 |
| CheckpointWriteErrorCount       | x    |       |        |                 |                 |
| EventPutAttemptCount            | x    | x     | x      | x               | x               |
| EventPutErrorCount              | x    |       |        |                 |                 |
| EventPutSuccessCount            | x    | x     | x      | x               | x               |
| EventTakeAttemptCount           | x    | x     | x      | x               | x               |
| EventTakeErrorCount             | x    |       |        |                 |                 |
| EventTakeSuccessCount           | x    | x     | x      | x               | x               |
| KafkaCommitTimer                |      | x     |        |                 |                 |
| KafkaEventGetTimer              |      | x     |        |                 |                 |
| KafkaEventSendTimer             |      | x     |        |                 |                 |
| Open                            | x    |       |        |                 |                 |
| RollbackCounter                 |      | x     |        |                 |                 |
| Unhealthy                       | x    |       |        |                 |                 |





## Ganglia

Flume支持Ganglia 3 or Ganglia 3.1格式的metrics。

启动时需要在flume-env.sh中配置flume.monitoring开头的属性，以向Ganglia服务发送metrics数据。

或者在启动时，添加相应参数

```shell
.bin/flume-ng agent --conf-file example.conf --name a1 -Dflume.monitoring.type=ganglia -Dflume.monitoring.hosts=com.example:1234,com.example2:5455
```



## JSON格式

Flume也可以发送JSON格式的metrics。同样只需要在flume-env.sh中配置属性，或启动时配置属性即可。

```shell
bin/flume-ng agent --conf-file example.conf --name a1 -Dflume.monitoring.type=http -Dflume.monitoring.port=34545
```





# Flume源码

## 启动流程







## Transaction

以MemoryTransaction为例，JdbcTransaction的原理跟数据库中的Transaction类似。

flume-ng-core下面有一个Transaction的interface，其包含了4个方法：



![image-20220218162501047](D:\工作文档\学习资料\bigdata_learning_materials\trainning_materials\Flume\md_pics\transactionInterface)

以及4个状态的枚举

```java
 enum TransactionState { Started, Committed, RolledBack, Closed }
```











