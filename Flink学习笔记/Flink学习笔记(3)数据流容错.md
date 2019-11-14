# 数据流容错

### 简介

Flink的容错机制保证即使出现了错误，数据流中的每条记录都只被reflect **exactly once**。Flink容错机制持续性地创建分布式数据流的snapshot，因为流应用程序有small state，所以这些snapshot都很轻量级，对性能影响很小。state存在一个可配置的地方(master node/HDFS) 。

出现程序错误的时候，Flink会停止分布式流数据流，然后系统会重启operator，reset到最近一次成功的checkpoint。输入流被重置到对应的状态snapshot，可以保证被重启的数据流里的记录都不在之前的checkpoint状态里。

*注意*：