# 1. Flink简介
## 1.1. 架构
**Apache Flink 擅长处理无界和有界数据集** 精确的时间控制和状态化使得 Flink 的运行时(runtime)能够运行任何处理无界流的应用。
提交或控制应用程序的所有通信都是通过 REST 调用进行的

## 1.2. 应用

### Streams
**有界** 和 **无界** 的数据流
**实时** 和 **历史记录** 的数据流

### State
Flink 提供了许多状态管理相关的特性支持:
-   **多种状态基础类型**：Flink 为多种不同的数据结构提供了相对应的状态基础类型，例如原子值（value），列表（list）以及映射（map）。
-   **插件化的State Backend**：State Backend 负责管理应用程序状态，并在需要的时候进行 checkpoint。Flink 支持多种 state backend，可以将状态存在内存或者 [RocksDB](https://rocksdb.org/)。RocksDB 是一种高效的嵌入式、持久化键值存储引擎。Flink 也支持插件式的自定义 state backend 进行状态存储。
-   **精确一次语义**：Flink 的 checkpoint 和故障恢复算法保证了故障发生后应用状态的一致性。
-   **超大数据量状态**：Flink 能够利用其异步以及增量式的 checkpoint 算法，存储数 TB 级别的应用状态。
-   **可弹性伸缩的应用**：Flink 能够通过在更多或更少的工作节点上对状态进行重新分布，支持有状态应用的分布式的横向伸缩。


### Time

-   **事件时间模式**：使用事件时间语义的流处理应用根据事件本身自带的时间戳进行结果的计算。
-   **Watermark 支持**：Flink 引入了 watermark 的概念，用以衡量事件时间进展。
-   **迟到数据处理**：当以带有 watermark 的事件时间模式处理数据流时，在计算完成之后仍会有相关数据到达。这样的事件被称为迟到事件。Flink 提供了多种处理迟到数据的选项，例如将这些数据重定向到旁路输出（side output）或者更新之前完成计算的结果。
-   **处理时间模式**：除了事件时间模式，Flink 还支持处理时间语义。处理时间模式根据处理引擎的机器时钟触发计算，一般适用于有着严格的低延迟需求，并且能够容忍近似结果的流处理应用.

### 分层API
Flink 根据抽象程度分层，提供了三种不同的 API。
![[api-stack.png]]
#### ProcessFunction
ProcessFunction 可以处理一或两条输入数据流中的单个事件或者归入一个特定窗口内的多个事件。

#### DataStream API
[DataStream API](https://nightlies.apache.org/flink/flink-docs-stable/dev/datastream_api.html)为许多通用的流处理操作提供了处理原语。这些操作包括窗口、逐条记录的转换操作，在处理事件时进行外部数据库查询等。DataStream API 支持 Java 和 Scala 语言，预先定义了例如`map()`、`reduce()`、`aggregate()` 等函数

#### SQL & Table API
Flink 支持两种关系型的 API，[Table API 和 SQL](https://nightlies.apache.org/flink/flink-docs-stable/dev/table/index.html)。这两个 API 都是批处理和流处理统一的 API，这意味着在无边界的实时数据流和有边界的历史记录数据流上，关系型 API 会以相同的语义执行查询，并产生相同的结果




## 1.3. 运维


# 2. Flink集群
## JobManager
JobManager 负责处理 [Job](https://nightlies.apache.org/flink/flink-docs-release-1.15/zh/docs/concepts/glossary/#flink-job) 提交、 Job 监控以及资源管理。

## Flink TaskManager
Flink TaskManager 运行 worker 进程， 负责实际任务 [Tasks](https://nightlies.apache.org/flink/flink-docs-release-1.15/zh/docs/concepts/glossary/#task) 的执行，而这些任务共同组成了一个 Flink Job。

# 3. Flink DataStream
Flink 中的 DataStream 是可以包含重复数据的**不可变**数据集。
Source创建DataStream, 基于该DataStream创建新的流。
## Flink程序剖析
Flink程序的基本组成：

* 获取执行环境（execution environment）  
* 读取数据源（source）  
* 定义基于数据的转换操作（transformations）  
* 定义计算结果的输出位置（sink）  
* 触发程序执行（execute）

### 创建执行环境


1.  获取一个`执行环境（execution environment）`；
```java
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
```
2. 加载/创建初始数据 （source）；
```java
DataStream<String> text = env.readTextFile("file:///path/to/file");
```
3. 指定数据相关的转换 （transformation）；
```java
DataStream<String> input = ...;

DataStream<Integer> parsed = input.map(new MapFunction<String, Integer>() {
    @Override
    public Integer map(String value) {
        return Integer.parseInt(value);
    }
});
```


4. 指定计算结果的存储位置（sink）；
```java
writeAsText(String path);

print();
```

5. 触发程序执行 （execute）。
```java
final JobClient jobClient = env.executeAsync();

final JobExecutionResult jobExecutionResult = jobClient.getJobExecutionResult().get();
```



# 4. Flink 部署

## Flink集群的基础架构
![[deployment_overview.svg]]


| 组件         | 用途                                           | 实现                                        |
| ------------ | ---------------------------------------------- | ------------------------------------------- |
| Flink Client | 代码转换为DAG，提交给JobManager                | 命令行、REST端口、SQL 客户端、 python客户端 |
| JobManager   | Flink集群的协调中心，负责分发work到TaskManager | Standalone、 Yarn、 K8S、Docker             |
| TasManager   | 实际执行任务的进程                             |                                             |


### JobManager的部署模式

![[deployment_modes.svg]]

#### Application Mode
> the `main()` method of the application is executed on the JobManager

main方法在JobManager上执行，每个应用对应一个集群

#### Per-Job Mode
每个Job一个集群，main方法在客户端运行。
无法采用standalone模式，只能在yarn或者k8s上采取此模式

#### Session Mode
先启动一个集群，保持一个session，客户端通过该session提交作业。
启动session时，集群资源便已经确定，所以提交的job都会争抢资源。

##  Standalone模式

### Session Mode
先start-cluster.sh，然后提交作业

### Application Mode
在  bin 目录下的 standalone-job.sh 来创建一个 JobManager
1. 执行以下命令，启动 JobManager。  
`$ ./bin/standalone-job.sh start --job-classname com.atguigu.wc.StreamWordCount  
这里我们直接指定作业入口类，脚本会到 lib 目录扫描所有的 jar 包。  
2. 同样是使用 bin 目录下的脚本，启动 TaskManager。  
`$ ./bin/taskmanager.sh start `
3. 如果希望停掉集群，同样可以使用脚本，命令如下。  
`$ ./bin/standalone-job.sh stop  
`$ ./bin/taskmanager.sh stop`


## Yarn 模式
### session mode
1. 启动集群
`./bin/yarn-session.sh -nm test`

可用参数解读：  
* -d：分离模式，如果你不想让 Flink YARN 客户端一直前台运行，可以使用这个参数，
即使关掉当前对话窗口，YARN session 也可以后台运行.
* -jm(--jobManagerMemory)：配置 JobManager 所需内存，默认单位 MB
* -nm(--name)：配置在 YARN UI 界面上显示的任务名。  
* -qu(--queue)：指定 YARN 队列名
* -tm(--taskManager)：配置每个 TaskManager 所使用内存。

2. 提交作业
* 通过 Web UI 提交作业
* 通过命令行提交作业
`$ bin/flink run -c com.atguigu.wc.StreamWordCount FlinkTutorial-1.0-SNAPSHOT.jar`
也可以通过`-m` 或者 `-jobmanager `参数指定 JobManager 的地址，JobManager 的地址在 YARN Session 的启动页面中可以找到。

### per-job mode

1. 通过命令行提交作业
`$ bin/flink run -d -t yarn-per-job -c com.atguigu.wc.StreamWordCount FlinkTutorial-1.0-SNAPSHOT.jar`

2. 在 YARN 的 ResourceManager 界面查看执行情况

3. 使用命令行查看或取消作业
`$ ./bin/flink list -t yarn-per-job -Dyarn.application.id=application_XXXX_YY`
`$ ./bin/flink cancel -t yarn-per-job -Dyarn.application.id=application_XXXX_YY <jobId> 

### application mode
1. 通过命令行提交作业
`$ bin/flink run-application -t yarn-application -c com.atguigu.wc.StreamWordCount FlinkTutorial-1.0-SNAPSHOT.jar

2. 通过命令行查看、取消作业
`./bin/flink list -t yarn-application -Dyarn.application.id=application_XXXX_YY  
`$ ./bin/flink cancel -t yarn-application  -Dyarn.application.id=application_XXXX_YY <jobId>

3. 通过 yarn.provided.lib.dirs 配置选项指定位置，将 jar 上传到远程
`./bin/flink run-application -t yarn-application -Dyarn.provided.lib.dirs="hdfs://myhdfs/my-remote-flink-dist-dir" hdfs://myhdfs/jars/my-application.jar

## K8S 模式



# 5. Flink 中的时间和窗口
## 5.1. WaterMark
### 5.1.1. WaterMark定义
* **事件时间（Event Time）** 数据产生时自带的时间，通常会嵌进数据中， Flink中的**WaterMark**表示**事件时间**的推进，是用来衡量事件时间进展的标记
* WaterMark是插入到数据流中的一个标记，可以认为是一个特殊数据
* 如果出现下游有多个并行子任务的情形，只要将WaterMark**广播**出去，就可以通知到所有下游任务当前的时间进度
* WaterMark随着事件时间单调递增
* 一个水位线 Watermark(t)，表示在当前流中事件时间已经达到了时间戳 t, 这代表 t 之前的所有数据都到齐了，之后流中不会出现时间戳 t’ ≤ t 的数据

有序流中的水位线 --- periodic WaterMark (根据系统时间定期插入)

乱序流中的水位线 ---  周期生成WaterMark的基础之上，加一个Lateness。该窗口内的最大事件时间减去Lateness作为WaterMark

### 5.1.2. 生成WaterMark
#### 5.1.2.1. 生成WaterMark的基本原则
* 尽量准确，尽量包含大部分迟到数据
* 单独创建一个Flink作业来监控事件流，建立概率分布或者机器学习模型，得到数据迟到的分布规律。
* [[迟到数据的处理方法]]

#### 5.1.2.2. WaterMarkStrategy
**DataStream API** ： `.assignTimestampsAndWatermarks(WatermarkStrategy watermarkStrategy)`
`.assignTimestampsAndWatermarks(WatermarkStrategy watermarkStrategy)`方法需要传入一个 WatermarkStrategy 作为参数
WatermarkStrategy 中包含 ：
* **TimestampAssigner**：主要负责从流中数据元素的某个字段中提取时间戳，并分配给**每条数据**。时间戳的分配是生成水位线的基础。
* **WatermarkGenerator** ： 主要负责按照既定的方式，基于时间戳生成水位线 。 在WatermarkGenerator 接口中，主要又有两个方法：onEvent()和 onPeriodicEmit()。
	* `onEvent`：每个事件（数据）到来都会调用的方法，它的参数有当前事件、时间戳以及允许发出水位线的一个 WatermarkOutput，可以基于事件做各种操作
	* `onPeriodicEmit`：周期性调用的方法，可以由 WatermarkOutput 发出水位线。周期时间为处理时间，可以调用环境配置的.setAutoWatermarkInterval()方法来设置，默认为200ms。


##### 内置WatermarkStrategy
* 单调递增有序流：`WatermarkStrategy.forMonotonousTimestamps()`
```
stream.assignTimestampsAndWatermarks(  
	WatermarkStrategy.<Event>forMonotonousTimestamps()
	.withTimestampAssigner(new SerializableTimestampAssigner<Event>()  
	{  
		@Override  
		public long extractTimestamp(Event element, long recordTimestamp)  
		{  
		return element.timestamp;  
		}  
	}));
```

* 乱序流： 需要设置一个固定的延迟时间（Lateness）
`WatermarkStrategy.forBoundedOutOfOrderness()

```
WatermarkStrategy.<Event>forBoundedOutOfOrderness(Duration.ofSeconds(5))
.withTimestampAssigner(new SerializableTimestampAssigner<Event>() {
		// 抽取时间戳的逻辑
		@Override
		public long extractTimestamp(Event element, long recordTimestamp) {
			return element.timestamp;
		}
}));
```

乱序流中生成的水位线真正的时间戳，其实是 `当前最大时间戳 – 延迟时间 – 1`，这里的单位是毫秒。 满足**前开后闭**区间

##### 自定义WatermarkStrategy
不同策略的关键就在于 WatermarkGenerator 的实现。整体说来，Flink  
有两种不同的生成水位线的方式：一种是周期性的（Periodic），另一种是断点式的（Punctuated）。

具体怎么结合使用，要看在哪个方法里发送水位线
`onEvent()` : 每个事件来调用
`onPeriodicEmit()` : 根据系统时间周期调用

```
public static class CustomPeriodicGenerator implements WatermarkGenerator<Event> {
	private Long delayTime = 5000L; // 延迟时间
	private Long maxTs = Long.MIN_VALUE + delayTime + 1L; // 观察到的最大时间戳
	
	@Override
	public void onEvent(Event event, long eventTimestamp, WatermarkOutput output) {
		// 每来一条数据就调用一次
		maxTs = Math.max(event.timestamp, maxTs); // 更新最大时间戳
	}
	@Override
	public void onPeriodicEmit(WatermarkOutput output) {
		// 发射水位线，默认 200ms 调用一次
		output.emitWatermark(new Watermark(maxTs - delayTime - 1L));
	}
}
```