# Flink学习笔记(6)流式分析

### Event Time和Watermark

##### 简介

Flink支持三种时间类型：

- event time：事件发生的时间
- ingestion time： Flink获得事件时所记录的时间戳
- processing time：特定的算子在pipeline中处理事件时的时间

基于processing time的计算分析会导致不一致，不利于重新分析历史数据或测试。

![img](https://upload-images.jianshu.io/upload_images/6178553-879bcae80f14c1bd.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

##### 处理event time

默认情况下，Flink使用process time。可以设置时间特征来更改：

```java
final StreamExecutionEnvironment env =
    StreamExecutionEnvironment.getExecutionEnvironment();
env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);
```

如果使用event time，还需要配合使用Timestamp Extractor和Watermark生成器，用来追踪event time的进度。

##### Watermark

带有时间戳的事件流有时候会发生顺序的混乱，如:

**···23 19 22 24 21 14 17 13 12 15 9 11 7 2 4→**

数字为实际发生时间，顺序为到达时间。假设需要构建一个流分类器，分好类的流按照时间戳排序。会有以下思考：

1. 流分类器看到的第一个元素是23，不能马上进行处理，因为可能是乱序的，有较早的元素没有到达。
2. 时间戳小于当前的元素有可能会抵达也有可能不会。
3. Watermark定义何时停止等待较早的事件
4. 生成Watermar：简单的方法是假定所有延迟受某个最大延迟的限制——有序无界Watermark。

##### 延时和完整性

批处理在产生结果之前可以完全了解输入，但流式传输需要在输入不完全的情况下产生结果。可以实施混合解决方案，刚开始的时候利用少量的数据快速产生初始结果，然后处理新数据时对初始结果提供更新。

##### 延迟

lateness相对于watermark定义，通过`Watermark(t)`可以判断时间`t`之前的数据都已经完成，`watermark(t)`之后的时间，且时间戳 `≤ t`是迟到的数据。

##### 使用Watermark

生成watermark的最简单方法是使用`WatermarkStrategy`：

```java
DataStream<Event> stream = ...

WatermarkStrategy<Event> strategy = WatermarkStrategy
        .<Event>forBoundedOutOfOrderness(Duration.ofSeconds(20))
        .withTimestampAssigner((event, timestamp) -> event.timestamp);

DataStream<Event> withTimestampsAndWatermarks =
    stream.assignTimestampsAndWatermarks(strategy);
```



### Windows 窗口

##### 简介

使用Flink计算窗口来分析依赖于两个抽象接口：`WindowAssigner`把事件分配给窗口，（根据需要创建新的窗口对象）；`WindowFunction`分配给窗口对时间进行计算。

Flink的Window API还有用于确定何时调用窗口函数的*Triggers*，可删除在窗口中收集的元素的*Evictors*。

在Keyed-Stream上使用窗口：

```java
stream.
    .keyBy(<key selector>)
    .window(<window assigner>)
    .reduce|aggregate|process(<window function>)
```

在Nonkeyed-Stream上使用Window：

```java
stream.
    .windowAll(<window assigner>)
    .reduce|aggregate|process(<window function>)
```

##### Window Assigners

Flink有几种内置的窗口分配器：

<img src="https://ci.apache.org/projects/flink/flink-docs-release-1.11/fig/window-assigners.svg" alt="窗口分配器" style="zoom: 67%;" />

- 分类
  - tumbling：滚动 
  - sliding：滑动
  - time：按时间
  - count：计数
  - session：按间隔来使用window
- 特点
  - 基于time/session
    - 无法正确处理历史数据
    - 无法正确处理乱序数据
    - 结果不确定
    - 延迟较低
  - 基于count
    - 完成批处理后Window才会被触发
  - 全局window
    - 将每个有相同key的时间分配给相同的全局窗口
    - 使用自定义触发器

##### Window Function

如何处理window内容？：

- 作为批处理，使用`ProcessWindowFunction`，把`Iterable`和窗口内容一起传递
  - 这种情况下，所有被分配到window的时间都会在keyed Flink状态里有缓存，内存消耗有时会很大
  - process里会传递一个Context，可以存储windowState，globalState，当处理下一个window时想要使用当前window的状态时可能会有用
- 每个event分配给窗口时，使用`ReduceFunction`或递增`AggregateFunction`调用。
- 前两者结合，将数据提前聚合，然后窗口触发时提供给`ProcessWindowFunction`。

##### 延迟事件

默认情况下，使用event time window时，迟到的event会被丢弃。Window API提供了两种控制方法：

- 可以把本来要丢弃的时间收集到备用输出流中，使用到的机制叫做Side Output。

  ```java
  OutputTag<Event> lateTag = new OutputTag<Event>("late"){};
  
  SingleOutputStreamOperator<Event> result = stream.
      .keyBy(...)
      .window(...)
      .sideOutputLateData(lateTag)
      .process(...);
    
  DataStream<Event> lateStream = result.getSideOutput(lateTag);
  ```

- 可以指定允许的延迟间隔，在此间隔内，延迟事件将继续分配给适当的窗口（其状态将被保留）。默认情况下，每个延迟事件都会导致再次调用window函数（有时称为*延迟触发*）。

  ```java
  stream.
      .keyBy(...)
      .window(...)
      .allowedLateness(Time.seconds(10))
      .process(...);
  ```

### 其他特性

- 滑动窗口可以创建许多窗口对象，并把每个event复制到相关的窗口里。
- 时间窗口和epoch对齐，通过可选的offset参数来调整。
- Windows可以接在另一个Windows后面。
- 空的TimeWindows不会有任何产出。
- 延迟的事件可能会导致延迟合并，session Windows基于间隔来进行合并窗口，起初每个事件都有一个单独窗口。

### 参考

https://ci.apache.org/projects/flink/flink-docs-release-1.11/zh/learn-flink/streaming_analytics.html

