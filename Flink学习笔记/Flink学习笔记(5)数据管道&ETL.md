# Flink学习笔记(5)数据管道&ETL

Apache Flink 的一种常见应用场景是 ETL（抽取、转换、加载）管道任务。

### 无状态转换

`map()` 和 `flatmap()`可以用于无状态转换。

##### `map()`

适用于一对一的转换，对于每个进入operator的流元素element，`map()`输出一个转换后的元素。

```java
//添加数据源
DataStream<TaxiRide> rides = env.addSource(new TaxiRideSource(...));
//定义map()函数
public static class Enrichment implements MapFunction<TaxiRide, EnrichedRide> {
    @Override
    public EnrichedRide map(TaxiRide taxiRide) throws Exception {
        return new EnrichedRide(taxiRide);
    }
}
//进行map()转换
DataStream<EnrichedRide> enrichedNYCRides = rides
    .filter(new RideCleansingSolution.NYCFilter())
    .map(new Enrichment());
```

##### `flatmap()`

使用接口中提供的 `Collector` ，`flatmap()` 可以输出你想要的任意数量的元素，也可以一个都不发。

```java
public static class NYCEnrichment implements FlatMapFunction<TaxiRide, EnrichedRide> {
//输出元素在out中
    @Override
    public void flatMap(TaxiRide taxiRide, Collector<EnrichedRide> out) throws Exception {
        FilterFunction<TaxiRide> valid = new RideCleansing.NYCFilter();
        if (valid.filter(taxiRide)) {
            out.collect(new EnrichedRide(taxiRide));
        }
    }
}
```

### Keyed Stream

##### `keyBy()`

Key的几种指定方式：

- **使用Tuple**来定义key：

  - `.keyBy(0)`
  - 嵌套：`.keyBy("f0.f1")`
  - 联合key：`keyBy(0,1)`，`keyBy("f0","f1")`
  - 编译器无法推断key的类型，Flink会把键值作为tuple传递。

- 可以**使用Field Expression**来指定key,一般作用的对象可以是<u>类对象</u>，或者<u>嵌套的Tuple格式</u>的数据。

  - 对于类对象可以使用类中的字段来指定key，类对象定义需要注意：
    - 类的访问级别必须是public
    - 必须写出默认的空的构造函数
    - 类中所有的字段必须是public的或者必须有getter，setter方法。
    - Flink必须支持字段的类型。

  - 对于嵌套的Tuple类型的Tuple数据可以使用"xx.f0"表示嵌套tuple中第一个元素，也可以直接使用”xx.0”来表示第一个元素。
  -  `.keyBy(value -> value.startCell)`

- **使用KeySelector**定义key：

  - ```java
    KeyedStream<String, String> keyBy = socketText.keyBy(new KeySelector<String, String>() {
                @Override
                public String getKey(String line) throws Exception {
                    return line.split("\t")[2];
                }
            });
    ```

- 通过**计算定义**key：
  - 可以按想要的方式计算得到键值，只要最终结果是确定的，并且实现了 `hashCode()` 和 `equals()`。
  - Tuple和POJO也可以作为key（满足上述条件）
  - `keyBy(ride -> GeoUtils.mapToGridCell(ride.startLon, ride.startLat))`

##### KeyedStream聚合

Flink 必须跟踪每个不同的键的最大时长。只要应用中有状态，你就应该考虑状态的大小。如果键值的数量是无限的，那 Flink 的状态需要的空间也同样是无限的。

在流处理场景中，考虑有限窗口的聚合往往比整个流聚合更有意义。

使用`reduce()`，`maxBy()`等对流进行聚合。

### 有状态转换

##### Rich Functions

`FilterFunction`， `MapFunction`，和 `FlatMapFunction`，这些都是单一抽象方法模式。

对其中的每一个接口，Flink 同样提供了一个所谓 “rich” 的变体，如 `RichFlatMapFunction`，其中增加了以下方法，包括：

- `open(Configuration c)`
- `close()`
- `getRuntimeContext()`

`open()` 仅在算子初始化时调用一次。可以用来加载一些静态数据，或者建立外部服务的链接等。

`getRuntimeContext()` 是你创建和访问 Flink 状态的途径。

例子：想象你有一个要去重的事件数据流，对每个键只保留第一个事件。

##### 清理状态

Flink 会对每个使用过的键都存储一个 `Boolean` 类型的实例。如果是键是有限的集合还好，但在键无限增长的应用中，清除再也不会使用的状态是很必要的。这通过在状态对象上调用 `clear()` 来实现，如下：

```
keyHasBeenSeen.clear()
```

### Connected Streams

有时你想要更灵活地调整转换的某些功能，比如数据流的阈值、规则或者其他参数。Flink 支持这种需求的模式称为 *connected streams* ，一个单独的算子有两个输入流。connected stream 也可以被用来实现流的关联。

<img src="https://ci.apache.org/projects/flink/flink-docs-release-1.11/fig/connected-streams.svg" alt="connected streams" style="zoom: 67%;" />

两个流只有键一致的时候才能连接。 `keyBy` 的作用是将流数据分区，当 keyed stream 被连接时，他们必须按相同的方式分区。`RichCoFlatMapFunction` 在状态中存了一个布尔类型的变量，这个变量被两个流共享。`RichCoFlatMapFunction` 是一种可以被用于一对连接流的 `FlatMapFunction`，并且它可以调用 rich function 的接口。这意味着它可以是有状态的。

### 参考

https://ci.apache.org/projects/flink/flink-docs-release-1.11/zh/learn-flink/etl.html

https://www.jianshu.com/p/faaa059453fb