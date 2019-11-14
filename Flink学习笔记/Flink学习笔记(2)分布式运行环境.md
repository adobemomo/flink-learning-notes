# Flink学习笔记（一）分布式运行环境

[TOC]



## Tasks和Operator Chains

### 什么是Operator Chains

 ![Operator chaining into Tasks](https://ci.apache.org/projects/flink/flink-docs-release-1.9/fig/tasks_chains.svg) 

如上图，在一个DataFlow上，每一个黄圈都算一个Task。Task可以被分为Subtask，Subtask分配在不同的节点上执行，再经shuffle然后`keyBy()`，整个Graph可以分布在不同的服务器上执行。



 ![img](http://img3.tbcdn.cn/5476e8b07b923/TB18Gv5JFXXXXcDXXXXXXXXXXXX) 

Chain可以把subtasks合并成一个task，每个task使用一个线程处理。Chaining Operator可以优化处理性能：

​    **1.**减少了线程切换之间的上下文转换开支（Task之间转切换即线程切换，合并subtask可以减少Task数量）。

​    **2.**提高了吞吐量并降低延迟。（Task内部减少了网络传输，不调用socket则速度变快）

​    **3.**减少消息的序列化和反序列化。（Task内部chain起来的subtask无需serialize和deserialize，直接传递对象）

​    **4.**减少了数据在缓冲区的交换（？）

### Operators Chain的内部实现

 ![img](http://img3.tbcdn.cn/5476e8b07b923/TB1cFbJJFXXXXaIXVXXXXXXXXXX) 

 如上图所示，Flink内部是通过`OperatorChain`这个类来将多个operator链在一起形成一个新的operator。`OperatorChain`形成的框框就像一个黑盒，Flink 无需知道黑盒中有多少个ChainOperator、数据在chain内部是怎么流动的，只需要将input数据交给 HeadOperator 就可以了，这就使得**`OperatorChain`在行为上与普通的operator无差别**，上面的OperaotrChain就可以看做是一个入度为1，出度为2的operator。所以在实现中，对外可见的只有HeadOperator，以及与外部连通的实线输出，这些输出对应了JobGraph中的JobEdge，在底层通过`RecordWriterOutput`来实现。另外，**框中的虚线是operator chain内部的数据流，这个流内的数据不会经过序列化/反序列化、网络传输**，而是直接将消息对象传递给下游的 ChainOperator 处理，这是性能提升的关键点。

## Job Managers，Task Managers和Clients

### JM，TM&Client的概念

Flink运行时会有两种进程，**JobManager**（负责调度Tasks，处理checkpoint和错误恢复, etc）和**TaskManager**（实际的SubTasks执行者）。至少有一个 Job manager，高可用性设置下会有多个JobManager，其中一个是leader，其他是standby。

JobManager和TaskManager有多种**启动方式**：standalone cluster, 容器，通过Yarn、Mesos等管理。TaskManager连接到JobManager，声明自己是可用的，然后JobManager会给他们分配任务。

**Client**不是运行时程序的一部分，但是会用来准备、send一个数据流到 JobManager。然后Client就可以断开连接，或者继续连着接收report。Client可以是以Java/Scala程序的形式触发execution，也可以是命令行进程`./bin/flink run ...`。

图中的Actor System是Flink底层akka中的实现分布式消息管理的，每个Actor是异步操作的最小原子。





​	 ![The processes involved in executing a Flink dataflow](https://ci.apache.org/projects/flink/flink-docs-release-1.9/fig/processes.svg) 

### Flink在Yarn上的部署

有两种方式，在Yarn上跑一个长期运行的Flink集群，或者提交一个Flink job到Yarn上。

##### 在YARN上开启一个长期运行的Flink集群

```bash
./bin/yarn-session.sh -jm 1024m -tm 4096m
```

`-s`可以用来指定每个Task Manager有几个slot。session开始后，可以通过`./bin/flink`向集群提交job。

##### 在YARN上运行一个Flink-job

```bash
./bin/flink run -m yarn-cluster -p 4 -yjm 1024m -ytm 4096m ./examples/batch/WordCount.jar
```

## Task Slots和资源

每个TaskManager都是一个JVM进程，TaskManager处理的**subtasks是线程粒度的**，一个TaskManager**有几个Slot就意味着它能支持几个并发线程**。Task Slot代表着TaskManager的固定资源子集，资源都是平均分配的。在同一个JVM上的Task共享TCP连接和心跳包message，也可能会共享一些data，避免了每个资源负载过重。

 ![A TaskManager with Task Slots and Tasks](https://ci.apache.org/projects/flink/flink-docs-release-1.9/fig/tasks_slots.svg) 

默认情况下，**Flink可以让来自不同Task的Subtask共享一个Task Slot**，只要是同一个job提交的即可。好处有二。其一：Flink集群需要的**Task Slot总数只要和job最高的并行度一样即可**，不需要计算一共有多少个Task；其二：**可以提高资源利用率**。如下图Source/map()使用资源并不那么intensive，如果像上面那样分配，S/m就会block掉一个线程，和keyBy()/window()这种对资源使用很intensive的子任务消耗了一样多的资源，简直unfair，换成下面这样，本来只有两个slot的任务可以分配到6个slot，可以避免heavy task的负载不均。

 ![TaskManagers with shared Task Slots](https://ci.apache.org/projects/flink/flink-docs-release-1.9/fig/slot_sharing.svg) 

一个好的默认Task Slot数应该是CPU内核数。

## State Backends & Savepoint

暂时觉得不重要，不做深入了解。



## 参考文献

[1] https://ci.apache.org/projects/flink/flink-docs-release-1.9/concepts/runtime.html#tasks-and-operator-chains

[2]  http://wuchong.me/blog/2016/05/09/flink-internals-understanding-execution-resources/ 

[3] https://blog.csdn.net/suifeng3051/article/details/49486927