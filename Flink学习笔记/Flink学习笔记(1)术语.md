# Flink学习笔记(2)术语

#### Flink应用程序集群

#### Flink Application Cluster

Flink应用程序集群是一个只执行单个Flink Job的专用[Flink集群](#Flink Cluster)。Flink集群的生命周期和[Flink Job](#Flink Job)的生命周期绑定。原来的Flink应用程序集群也就是大家所知的job模式下的Flink集群。与[Flink会话集群](#Flink Session Cluster)进行比较。



#### Flink集群

#### Flink Cluster

（典型地）由一个[Flink Master/Flink主节点](#Flink Master)和一个或多个[Flink TaskManager/Flink任务管理器](#Flink TaskManager)进程组成的分布式系统。



#### 事件

#### Event

事件是对由应用建模的域状态更改的声明。事件可以是流或者是批处理应用的输入和输出。事件是特殊类型的[记录](#Record)。



#### 执行图

#### ExecutionGraph

请查看[物理图](#Physical Graph)。



#### 函数

#### Function

函数由用户实现，封装了Flink程序的应用逻辑。大多数函数由相应的[操作符](#Operator)包装。



#### 实例

#### Instance

实例这个术语被用来描述运行时一个特定类型的特定实例（通常是[操作符](#Operator)或者[函数](#Function)）。由于Apache Flink主要是用Java编写的，因此它对应于Java中的实例/Instance或对象/Object的定义。在Apache Flink的上下文中，术语*并行实例*也经常被用于强调有多个相同[操作符](#Operator)或者[函数](#Function)类型的示例正在并行运行。



#### Flink 作业

#### Flink Job

Flink Job是Flink程序的运行时表示。Flink Job既可以被提交到长期运行的[Flink会话集群](#Flink Session Cluster)，也可以作为独立的[Flink应用集群](#Flink应用程序集群/Flink Application Cluster)来启动。



#### 作业图

#### JobGraph

请查看[逻辑图](#Logical Graph)。



#### Flink作业管理器

#### Flink JobManager

任务管理器是运行在[Flink Master/Flink主节点](#Flink Master)上的组件之一。负责监督单个Job/作业的[Task/任务](#Task)执行。以前一整个[Flink Master/Flink主节点](#Flink Master)被叫做JobManager/作业管理器。



#### 逻辑图

#### Logical Graph

逻辑图是描述流处理程序高层逻辑的有向图。节点是[操作符](#Operator)，边指示了输入/输出关系，或者是数据流/数据集。



#### 受管状态

#### Managed State

受管状态描述了已经在框架中注册的应用程序状态。对于受管状态，Apache Flink特别关注它的持久性和可伸缩性。



#### Flink主节点

#### Flink Master

Flink主节点是一个[Flink集群](#Flink Cluster)的master，包括三个不同的组件：Flink资源管理器，Flink分派器和[Flink作业管理器](#Flink JobManager)（每个作业管理器运行一个[Flink Job](#Flink Job)）。



#### 操作符

#### Operator

[逻辑图](#Logical Graph)的节点。操作符实现某种操作，通常由[函数](#Function)来执行。源和接收器是用于数据接收和数据出口的特殊操作符。



#### 操作符链

#### Operator Chain

操作符链由两个或者两个以上的的连续[操作符](#Operator)组成，中间没有任何重新分区。在同一个操作符链中的操作符无需经过序列化或者Flink网络堆栈就可以彼此之间转发记录。



#### 分区

#### Partition

分区是整个数据流或数据集的独立子集。通过把每个[记录](#Record)分配到一个或多个分分区，把数据流/数据集分成不同的分区。数据流/数据集的分区在运行时被[任务](#Task)所消费。改变数据流或数据集分区方式的转换通常称为重新分区。



#### 物理图

#### Physical Graph

物理图是[逻辑图](#Logical Graph)在分布式运行时的转换结果。节点为[任务](#Task)，边指示输入/输出关系，或者是数据流/数据集的[分区](#Partition)。



#### 记录

#### Record

记录是数据流/数据集的组成元素。[操作符](#Operator)和[函数](#Function)接收记录作为输入，提交记录作为输出。



#### Flink会话集群

#### Flink Session Cluster 

Flink会话集群是可以长期接收多个待执行Flink Job的[Flink集群](#Flink Cluster)。集群的生命周期不和任何一个Flink Job的生命周期绑定。以前Flink会话集群以session模式下的Flink集群被大家熟知。与[Flink应用程序集群](#Flink Application Cluster)进行比较。



#### 状态后端

#### State Backend

对于流处理程序，[Flink Job](#Flink Job)的状态后端决定了Job[状态]是如何存储在每个任务管理器上的（任务管理器的Java堆或者（嵌入式）RocksDB），也决定了Job状态写在检查点的哪里（Flink主节点的Java堆或者文件系统）。



#### 子任务

#### Sub-Task

子任务是负责处理数据流[分区](#Partition)的[任务](#Task)。术语“子任务”强调他们是由相同的[操作符](#Operator)或[操作符链](#Operator Chain)所处理的多个并发任务。



#### 任务

#### Task

[物理图](#Physical Graph)的节点。任务是工作的基本单元，由Flink运行时来执行。任务封装恰好一个[操作符](#Operator)或者[操作符链](#Operator Chain)的并行实例。



#### Flink任务管理器

#### Flink TaskManager

任务管理器是[Flink集群](#Flink Cluster)的从节点。[任务](#Task)被调度到任务管理器上执行。他们彼此通信以在接在来的任务之间交换数据。



#### 转换

#### Transformation

转换作用于一个或多个数据集/数据流，输出一个或多个数据集/数据流。转换可能会在单个记录的基础上改变数据集/数据流，也可能只改变它的分区，或者进行聚集。尽管[操作符](#Operator)和[函数](#Function)是Flink API的“物理”部分，转换只是一个API概念。特殊地，大部分，但不是全部，由特定[操作符](#Operator)实现。