## Demo Applications for Apache Flink&trade; DataStream

This repository contains demo applications for [Apache Flink](https://flink.apache.org)'s
[DataStream API](https://ci.apache.org/projects/flink/flink-docs-release-0.10/apis/streaming_guide.html).

Apache Flink is a scalable open-source streaming dataflow engine with many competitive features. <br>
You can find a list of Flink's features at the bottom of this page.

### Run a demo application in your IDE

You can run all examples in this repository from your IDE and play around with the code. <br>
Requirements:

- Java JDK 7 (or 8)
- Apache Maven 3.x
- Git
- an IDE with Scala support (we recommend IntelliJ IDEA)

To run a demo application in your IDE follows these steps:

1. **Clone the repository:** Open a terminal and clone the repository:
`git clone https://github.com/dataArtisans/flink-streaming-demo.git`. Please note that the
repository is about 100MB in size because it includes the input data of our demo applications.

2. **Import the project into your IDE:** The repository is a Maven project. Open your IDE and
import the repository as an existing Maven project. This is usually done by selecting the folder that
contains the `pom.xml` file or selecting the `pom.xml` file itself.

3. **Start a demo application:** Execute the `main()` method of one of the demo applications, for example
`com.dataartisans.flink_demo.examples.TotalArrivalCount.scala`.
Running an application will start a local Flink instance in the JVM process of your IDE.
You will see Flink's log messages and the output produced by the program being printed to the standard output.

4. **Explore the web dashboard:** The local Flink instance starts a webserver that serves Flink's
dashboard. Open [http://localhost:8081](http://localhost:8081) to access and explore the dashboard.

### Demo applications

#### Taxi event stream

All demo applications in this repository process a stream of taxi ride events that
originate from a [public data set](http://www.nyc.gov/html/tlc/html/about/trip_record_data.shtml)
of the [New York City Taxi and Limousine Commission](http://www.nyc.gov/html/tlc/html/home/home.shtml)
(TLC). The data set consists of records about taxi trips in New York City from 2009 to 2015.

We took some of this data and converted it into a data set of taxi ride events by splitting each
trip record into a ride start and a ride end event. The events have the following schema:

```
rideId: Long // unique id for each ride
time: DateTime // timestamp of the start/end event
isStart: Boolean // true = ride start, false = ride end
location: GeoPoint // lon/lat of pick-up/drop-off location
passengerCnt: short // number of passengers
travelDist: float // total travel distance, -1 on start events
```

A custom `SourceFunction` serves a `DataStream[TaxiRide]` from this data set.
In order to generate the stream as realistically as possible, events are emitted according to their
timestamp. Two events that occurred ten minutes after each other in reality are served ten minutes apart.
A speed-up factor can be specified to "fast-forward" the stream, i.e., with a speed-up factor of 2,
the events would be served five minutes apart. Moreover, you can specify a maximum serving delay
which causes each event to be randomly delayed within the bound to simulate an out-of-order stream
(a delay of 0 seconds results in an ordered stream). All examples operate in event-time mode.
This guarantees consistent results even in case of historic data or data which is delivered out-of-order.

#### Identify popular locations

The [`TotalArrivalCount.scala`](/src/main/scala/com/dataartisans/flink_demo/examples/TotalArrivalCount.scala)
program identifies popular locations in New York City.
It ingests the stream of taxi ride events and counts for each location the number of persons that
arrive by taxi.

#### Identify the popular locations of the last 15 minutes

The [`SlidingArrivalCount.scala`](/src/main/scala/com/dataartisans/flink_demo/examples/SlidingArrivalCount.scala)
program identifies popular locations of the last 15 minutes.
It ingests the stream of taxi ride records and computes every five minutes the number of
persons that arrived at each location within the last 15 minutes.
This type of computation is known as sliding window.


#### Compute early arrival counts for popular locations

Some stream processing use cases depend on timely event aggregation, for example to send out notifications or alerts.
The [`EarlyArrivalCount.scala`](/src/main/scala/com/dataartisans/flink_demo/examples/EarlyArrivalCount.scala)
program extends our previous sliding window application. Same as before, it computes every five minutes
the number of persons that arrived at each location within the last 15 minutes.
In addition it emits an early partial count whenever a multitude of 50 persons arrived at a
location, i.e., it emits an updated count if more than 50, 100, 150 (and so on) persons arrived at a location.

### Setting up Elasticsearch and Kibana

The demo applications in this repository are prepared to write their output to [Elasticsearch](https://www.elastic.co/products/elasticsearch).
Data in Elasticsearch can be easily visualized using [Kibana](https://www.elastic.co/products/kibana)
for real-time monitoring and interactive analysis.

Our demo applications depend on Elasticsearch 1.7.3 and Kibana 4.1.3. Both systems have a nice
out-of-the-box experience and operate well with their default configurations for our purpose.

Follow these instructions to set up Elasticsearch and Kibana.

#### Setup Elasticsearch

1. Download Elasticsearch 1.7.3 [here](https://www.elastic.co/downloads/past-releases/elasticsearch-1-7-3).

1. Extract the downloaded archive file and enter the extracted repository.

1. Start Elasticsearch using the start script: `./bin/elasticsearch`.

1. Create an index (here called `nyc-idx`): `curl -XPUT "http://localhost:9200/nyc-idx"`

1. Create a schema mapping for the index (here called `popular-locations`):
 ```
 curl -XPUT "http://localhost:9200/nyc-idx/_mapping/popular-locations" -d'
 {
  "popular-locations" : {
    "properties" : {
      "cnt": {"type": "integer"},
      "location": {"type": "geo_point"},
      "time": {"type": "date"}
    }
  }
 }'
 ```
 **Note:** This mapping can be used for all demo application.
1. Configure a demo application to write its results to Elasticsearch. For that you have to change the corresponding parameters in the demo applications source code:
  - set `writeToElasticsearch = true`
  - set `elasticsearchHost` to the correct host name (see Elasticsearch's log output)

1. Run the Flink program to write its result to Elasticsearch.

To clear the `nyc-idx` index in Elasticsearch, simply drop the mapping as
`curl -XDELETE 'http://localhost:9200/nyc-idx/popular-locations'` and create it again with the previous
command.

#### Setup Kibana

Setting up Kibana and visualizing data that is stored in Elasticsearch is also easy.

1. Dowload Kibana 4.1.3 [here](https://www.elastic.co/downloads/past-releases/kibana-4-1-3)

1. Extract the downloaded archive and enter the extracted repository.

1. Start Kibana using the start script: `./bin/kibana`.

1. Access Kibana by opening [http://localhost:5601](http://localhost:5601) in your browser.

1. Configure an index pattern by entering the index name "nyc-idx" and clicking on "Create".
Do not uncheck the "Index contains time-based events" option.

1. Click on the "Discover" button at the top of the page. Kibana will tell you "No results found" 
because we have to configure the time range of the data to visualize in Kibane. Click on the 
"Last 15 minutes" label in the top right corner and enter an absolute time range from 2013-01-01 
to 2013-01-06 which is the time range of our taxi ride data stream. You can also configure a 
refresh interval to reload the page for updates.

1. Click on the “Visualize” button at the top of the page, select "Tile map", and click on "From a
new search".

1. Next you need to configure the tile map visualization:

  - Top-left: Configure the displayed value to be a “Sum” aggregation over the "cnt" field.
  - Top-left: Select "Geo Coordinates" as bucket type and make sure that "location" is
configured as field.
  - Top-left: You can change the visualization type by clicking on “Options” (top left) and selecting
for example a “Shaded Geohash Grid” visualization.
  - The visualization is started by clicking on the green play button.

The following screenshot shows how Kibana visualizes the result of `TotalArrivalCount.scala`.

![Kibana Screenshot](/data/kibana.jpg?raw=true "Kibana Screenshot")

### Apache Flink's Feature Set

- **Support for out-of-order streams and event-time processing**: In practice, streams of events rarely
arrive in the order that they are produced, especially streams from distributed systems, devices, and sensors.
Flink 0.10 is the first open source engine that supports out-of-order streams and event
time which is a hard requirement for many application that aim for consistent and meaningful results.

- **Expressive and easy-to-use APIs in Scala and Java**: Flink's DataStream API provides many
operators which are well known from batch processing APIs such as `map`, `reduce`, and `join` as
well as stream specific operations such as `window`, `split`, and `connect`.
First-class support for user-defined functions eases the implementation of custom application
behavior. The DataStream API is available in Scala and Java.

- **Support for sessions and unaligned windows**: Most streaming systems have some concept of windowing,
i.e., a temporal grouping of events based on some function of their timestamps. Unfortunately, in
many systems these windows are hard-coded and connected with the system’s internal checkpointing
mechanism. Flink is the first open source streaming engine that completely decouples windowing from
fault tolerance, allowing for richer forms of windows, such as sessions.

- **Consistency, fault tolerance, and high availability**: Flink guarantees consistent operator state
in the presence of failures (often called "exactly-once processing"), and consistent data movement
between selected sources and sinks (e.g., consistent data movement between Kafka and HDFS). Flink
also supports master fail-over, eliminating any single point of failure.

- **High throughput and low-latency processing**: We have clocked Flink at 1.5 million events per second per core,
and have also observed latencies at the 25 millisecond range in jobs that include network data
shuffling. Using a tuning knob, Flink users can control the latency-throughput trade-off, making
the system suitable for both high-throughput data ingestion and transformations, as well as ultra
low latency (millisecond range) applications.

- **Integration with many systems for data input and output**: Flink integrates with a wide variety of
open source systems for data input and output (e.g., HDFS, Kafka, Elasticsearch, HBase, and others),
deployment (e.g., YARN), as well as acting as an execution engine for other frameworks (e.g.,
Cascading, Google Cloud Dataflow). The Flink project itself comes bundled with a Hadoop MapReduce
compatibility layer, a Storm compatibility layer, as well as libraries for Machine Learning and
graph processing.

- **Support for batch processing**: In Flink, batch processing is a special case of stream processing,
as finite data sources are just streams that happen to end. Flink offers a dedicated execution mode
for batch processing with a specialized DataSet API and libraries for Machine Learning and graph processing. In
addition, Flink contains several batch-specific optimizations (e.g., for scheduling, memory
management, and query optimization), matching and even out-performing dedicated batch processing
engines in batch use cases.

- **Developer productivity and operational simplicity**: Flink runs in a variety of environments. Local
execution within an IDE significantly eases development and debugging of Flink applications.
In distributed setups, Flink runs at massive scale-out. The YARN mode
allows users to bring up Flink clusters in a matter of seconds. Flink serves monitoring metrics of
jobs and the system as a whole via a well-defined REST interface. A build-in web dashboard
displays these metrics and makes monitoring of Flink very convenient.


<hr>

<center>
Copyright &copy; 2015 dataArtisans. All Rights Reserved.

Apache Flink, Apache, and the Apache feather logo are trademarks of The Apache Software Foundation.
</center>
Apache Flink™DataStream的演示应用程序
该存储库包含Apache Flink的 DataStream API的演示应用程序。

Apache Flink是一个可扩展的开源流数据流引擎，具有许多竞争功能。
您可以在本页底部找到Flink功能列表。

在IDE中运行演示应用程序
您可以从IDE运行此存储库中的所有示例，并使用代码。
要求：

Java JDK 7（或8）
Apache Maven 3.x
混帐
支持Scala的IDE（我们推荐使用IntelliJ IDEA）
要在IDE中运行演示应用程序，请执行以下步骤：

克隆存储库：打开终端并克隆存储库： git clone https://github.com/dataArtisans/flink-streaming-demo.git。请注意，存储库大小约为100MB，因为它包含我们的演示应用程序的输入数据。

将项目导入IDE：存储库是Maven项目。打开IDE并将存储库导入为现有Maven项目。这通常通过选择包含pom.xml文件的文件夹或选择pom.xml文件本身来完成。

启动演示应用程序：例如，执行其中一个演示应用程序的main()方法 com.dataartisans.flink_demo.examples.TotalArrivalCount.scala。运行应用程序将在IDE的JVM进程中启动本地Flink实例。您将看到Flink的日志消息以及由打印到标准输出的程序生成的输出。

浏览Web仪表板：本地Flink实例启动一个为Flink仪表板提供服务的Web服务器。打开http：// localhost：8081以访问和浏览仪表板。

演示应用程序
出租车事件流
在这个过程中存储库的所有演示应用程序是从起源乘坐出租车事件流的公共数据集 的的纽约市出租车和轿车委员会 （TLC）。该数据集包括2009年至2015年纽约市出租车旅行的记录。

我们将这些数据中的一部分转换为出租车事件的数据集，将每个行程记录分成骑行开始和骑行结束事件。事件具有以下架构：

rideId: Long // unique id for each ride
time: DateTime // timestamp of the start/end event
isStart: Boolean // true = ride start, false = ride end
location: GeoPoint // lon/lat of pick-up/drop-off location
passengerCnt: short // number of passengers
travelDist: float // total travel distance, -1 on start events
自定义SourceFunction服务DataStream[TaxiRide]来自此数据集。为了尽可能逼真地生成流，事件根据其时间戳发出。实际上相隔10分钟发生的两个事件相隔十分钟。可以指定加速因子以“快进”流，即，加速因子为2，事件将相隔五分钟服务。此外，您可以指定最大服务延迟，这会导致每个事件在绑定范围内随机延迟，以模拟无序流（延迟0秒会产生有序流）。所有示例都在事件时间模式下运行。即使在历史数据或无序传送的数据的情况下，这也可以保证一致的结果。

确定热门位置
该TotalArrivalCount.scala 计划确定了纽约市的热门地点。它摄取了出租车活动的流程，并计算每个地点乘出租车到达的人数。

确定过去15分钟的热门位置
该SlidingArrivalCount.scala 计划确定了过去15分钟的热门位置。它摄取出租车记录流并每五分钟计算在过去15分钟内到达每个位置的人数。这种类型的计算称为滑动窗口。

计算热门地点的提前到达计数
某些流处理用例取决于及时的事件聚合，例如发送通知或警报。该EarlyArrivalCount.scala 程序扩展了之前的滑动窗口应用程序。与之前相同，它每五分钟计算一次在过去15分钟内到达每个地点的人数。此外，每当有50人到达某个位置时，它会发出早期的部分计数，即如果超过50,100,150（等等）人员到达某个位置，则会发出更新的计数。

设置Elasticsearch和Kibana
此存储库中的演示应用程序准备将其输出写入Elasticsearch。使用Kibana可以轻松地将Elasticsearch中的数据可视化，以 进行实时监控和交互式分析。

我们的演示应用程序依赖于Elasticsearch 1.7.3和Kibana 4.1.3。这两个系统都具有良好的开箱即用体验，并且可以根据我们的目的很好地运行其默认配置。

按照以下说明设置Elasticsearch和Kibana。

设置Elasticsearch
在这里下载Elasticsearch 1.7.3 。

提取下载的存档文件并输入解压缩的存储库。

使用启动脚本启动Elasticsearch : ./bin/elasticsearch.

创建索引（此处称为nyc-idx）：curl -XPUT "http://localhost:9200/nyc-idx"

为索引创建模式映射（此处称为popular-locations）：

curl -XPUT "http://localhost:9200/nyc-idx/_mapping/popular-locations" -d'
{
 "popular-locations" : {
   "properties" : {
     "cnt": {"type": "integer"},
     "location": {"type": "geo_point"},
     "time": {"type": "date"}
   }
 }
}'
注意：此映射可用于所有演示应用程序。

配置演示应用程序以将其结果写入Elasticsearch。为此，您必须更改演示应用程序源代码中的相应参数：
组 writeToElasticsearch = true
设置elasticsearchHost为正确的主机名（请参阅Elasticsearch的日志输出）
运行Flink程序将其结果写入Elasticsearch。
要清除nyc-idxElasticsearch中的索引，只需删除映射， curl -XDELETE 'http://localhost:9200/nyc-idx/popular-locations'然后使用上一个命令再次创建它。

设置Kibana
设置Kibana并可视化存储在Elasticsearch中的数据也很容易。

在这里下载 Kibana 4.1.3

提取下载的存档并输入提取的存储库。

使用启动脚本启动Kibana : ./bin/kibana.

通过在浏览器中打开http：// localhost：5601来访问Kibana 。

通过输入索引名称“nyc-idx”并单击“创建”来配置索引模式。不要取消选中“索引包含基于时间的事件”选项。

单击页面顶部的“发现”按钮。Kibana将告诉您“未找到结果”，因为我们必须配置数据的时间范围以在Kibane中可视化。单击右上角的“最后15分钟”标签，输入2013-01-01至2013-01-06的绝对时间范围，这是我们的出租车数据流的时间范围。您还可以配置刷新间隔以重新加载页面以进行更新。

单击页面顶部的“可视化”按钮，选择“平铺地图”，然后单击“从新搜索”。

接下来，您需要配置切片贴图可视化：

左上角：将显示的值配置为“cnt”字段上的“Sum”聚合。
左上角：选择“地理坐标”作为铲斗类型，并确保将“位置”配置为字段。
左上角：您可以通过单击“选项”（左上角）并选择例如“着色Geohash网格”可视化来更改可视化类型。
通过单击绿色播放按钮启动可视化。
以下屏幕截图显示了Kibana如何可视化结果TotalArrivalCount.scala。

Kibana截图

Apache Flink的功能集
支持无序流和事件时间处理：实际上，事件流很少按照生成的顺序到达，尤其是来自分布式系统，设备和传感器的流。Flink 0.10是第一个支持无序流和事件时间的开源引擎，这是许多旨在获得一致且有意义的结果的应用程序的硬性要求。

在Scala和Java的表现和易于使用的API：弗林克的的数据流中的API提供了很好的批量处理的API已知的，诸如许多运营商map，reduce以及join如以及流特定的操作window，split和connect。对用户定义函数的一流支持简化了自定义应用程序行为的实现。DataStream API在Scala和Java中可用。

支持会话和未对齐窗口：大多数流式传输系统具有一些窗口化概念，即基于其时间戳的某些功能的事件的时间分组。不幸的是，在许多系统中，这些窗口都是硬编码的，并与系统的内部检查点机制相连。Flink是第一个完全将窗口与容错分离的开源流引擎，允许更丰富的窗口形式，例如会话。

一致性，容错性和高可用性：Flink在出现故障时保证一致的操作员状态（通常称为“精确一次处理”），并在选定的源和接收器之间保持一致的数据移动（例如，Kafka和HDFS之间的一致数据移动） 。Flink还支持主故障转移，消除任何单点故障。

高吞吐量和低延迟处理：我们为每个核心每秒150万个事件计时Flink，并且还在包含网络数据混洗的作业中观察到25毫秒范围内的延迟。使用调谐旋钮，Flink用户可以控制延迟 - 吞吐量权衡，使系统适用于高吞吐量数据摄取和转换，以及超低延迟（毫秒范围）应用。

与许多数据输入和输出系统集成：Flink集成了各种用于数据输入和输出的开源系统（例如，HDFS，Kafka，Elasticsearch，HBase等），部署（例如，YARN），以及充当其他框架的执行引擎（例如，Cascading，Google Cloud Dataflow）。Flink项目本身捆绑了Hadoop MapReduce兼容层，Storm兼容层，以及用于机器学习和图形处理的库。

支持批处理：在Flink中，批处理是流处理的一种特殊情况，因为有限数据源只是即将结束的流。Flink提供专用的批处理执行模式，使用专门的DataSet API和用于机器学习和图形处理的库。此外，Flink包含多个特定于批处理的优化（例如，用于调度，内存管理和查询优化），在批处理用例中匹配甚至超出专用批处理引擎。

开发人员的工作效率和操作简便性：Flink可在各种环境中运行。IDE中的本地执行显着简化了Flink应用程序的开发和调试。在分布式设置中，Flink以大规模横向扩展运行。YARN模式允许用户在几秒钟内启动Flink集群。Flink通过定义良好的REST接口提供作业和整个系统的监控指标。内置Web仪表板显示这些指标，使Flink的监控非常方便。
