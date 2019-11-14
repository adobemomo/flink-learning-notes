# Flink REST API

REST API是版本化的，使用的时候需要把版本号加到前缀里。前缀的格式为 `v[version_number]` 。如果想要获得`/foo/bar`的version 1，需要请求 `/v1/foo/bar`。

在linux命令行下可以使用`curl`进行请求。

e.g. `curl -X GET http://0.0.0.0:8081/v1/jobs/`

ip为flink-conf.yaml中的`rest.ip`

port为flinkconf.yaml中的`rest.port`

**以下Response省略**

## V1

### /cluster

#####  关闭集群

| **<u>/cluster</u>** |                         |
| :------------------ | :---------------------- |
| `DELETE`            | Response code: `200 OK` |
| 关闭集群            |                         |
| **Request**         |                         |
| `{}`                |                         |
| **Response**        |                         |
| `{}`                |                         |

## /config

##### 返回Web UI配置

| **<u>/config</u>** |                         |
| ------------------ | ----------------------- |
| `GET`              | Response code: `200 OK` |
| 返回Web UI的配置   |                         |
| **Request**        |                         |
| **Response**       |                         |

## /jars

##### 返回jar列表

| **/jars**                                     |                         |
| --------------------------------------------- | ----------------------- |
| `GET`                                         | Response code: `200 OK` |
| 返回之前通过'/jars/upload'上传的所有jar列表。 |                         |
| **Request**                                   |                         |
| **Response**                                  |                         |

##### 上传jar

| **/jars/upload**                                             |                         |
| ------------------------------------------------------------ | ----------------------- |
| `POST`                                                       | Response code: `200 OK` |
| 上传jar到集群。jar必须作为多部份数据发送。鉴于一些http库不会默认添加header，确保"Content-Type" header被设置为"application/x-java-archive"。可以通过`curl -X POST -H "Expect:" -F "jarfile=@path/to/flink-job.jar" http://hostname:port/jars/upload`来使用curl提交jar。 |                         |
| **Request**                                                  |                         |
| **Response**                                                 |                         |

#### /jars/:jarid

##### 删除jar

| **/jars/:jarid**                                             |                         |
| ------------------------------------------------------------ | ----------------------- |
| `DELETE`                                                     | Response code: `200 OK` |
| 删除之前通过'/jars/upload'上传的jar。                        |                         |
| **路径参数**                                                 |                         |
| `jarid` - 识别jar的字符串值。当上传jar时会返回一个路径，文件名就是这个ID。这个值和已上传jar列表里的id字段相同(/jars)。 |                         |
| **Request**                                                  |                         |
| **Response**                                                 |                         |

##### 获取jar提交job的plan

| **/jars/:jarid/plan**                                        |                         |
| ------------------------------------------------------------ | ----------------------- |
| `GET`                                                        | Response code: `200 OK` |
| 返回一个包含在通过'/jars/upload'上传jar中的job的数据流plan。程序参数既可以通过JSON请求传递（推荐），也可以通过query参数传递。 |                         |
| 路径参数                                                     |                         |
| `jarid` - 识别jar的字符串值。当上传jar时会返回一个路径，文件名就是这个ID。这个值和已上传jar列表里的id字段相同(/jars)。 |                         |
| **Query 参数**                                               |                         |
| --`program-args` (可选):已经被废弃，请使用'programArg'来代替。为程序或者plan指定参数的字符串值。 |                         |
| --`programArg` (可选): 用逗号分割的程序参数。                |                         |
| --`entry-class` (可选): 字符串值，用于指定入口类的全限定名称。会覆盖jar文件manifest中定义的类。 |                         |
| --`parallelism` (可选): 正整数值，指定想要的job并行度。      |                         |
| **Request**                                                  |                         |
| **Response**                                                 |                         |

##### 通过jar提交job

| **/jars/:jarid/run**                                         |                         |
| ------------------------------------------------------------ | ----------------------- |
| `POST`                                                       | Response code: `200 OK` |
| 通过运行via'/jars/upload'上传的jar提交一个job。程序参数既可以通过JSON请求传递（推荐），也可以通过query参数传递。 |                         |
| **路径参数**                                                 |                         |
| `jarid` - 识别jar的字符串值。当上传jar时会返回一个路径，文件名就是这个ID。这个值和已上传jar列表里的id字段相同(/jars)。 |                         |
| **Query 参数**                                               |                         |
| --`allowNonRestoredState` (可选): 布尔值，如果保存点包括了不能被映射回job的状态，用来指定是否应该拒绝job提交。 |                         |
| --`savepointPath` (可选): 字符串值，指定要从中还原job的保存点的路径。 |                         |
| --`program-args` (可选): 已被废弃，请使用'programArg'来代替。字符串值，为程序或者plan指定参数。 |                         |
| --`programArg` (可选): 用逗号分割的程序参数。                |                         |
| --`entry-class` (可选): 字符串值，用于指定入口类的全限定名称。会覆盖jar文件manifest中定义的类。 |                         |
| --`parallelism` (可选): 正整数值，指定想要的job并行度。      |                         |
| **Request**                                                  |                         |
| {   "type" : "object",   "id" : "urn:jsonschema:org:apache:flink:runtime:webmonitor:handlers:JarRunRequestBody",   "properties" : {     "entryClass" : {       "type" : "string"     },     "programArgs" : {       "type" : "string"     },     "programArgsList" : {       "type" : "array",       "items" : {         "type" : "string"       }     },     "parallelism" : {       "type" : "integer"     },     "jobId" : {       "type" : "any"     },     "allowNonRestoredState" : {       "type" : "boolean"     },     "savepointPath" : {       "type" : "string"     }   } } |                         |
| **Response**                                                 |                         |

## /jobmanager

##### 返回集群配置

| **/jobmanager/config** |                         |
| ---------------------- | ----------------------- |
| `GET`                  | Response code: `200 OK` |
| 返回集群配置。         |                         |
| **Request**            |                         |
| **Response**           |                         |

##### 访问job manager指标

| **/jobmanager/metrics**                                  |                         |
| -------------------------------------------------------- | ----------------------- |
| `GET`                                                    | Response code: `200 OK` |
| 提供对job管理器指标的访问。                              |                         |
| **Query 参数**                                           |                         |
| `get` (可选): 用逗号分隔，用于选择特定指标的字符串列表。 |                         |
| Request                                                  |                         |
| Response                                                 |                         |

## /jobs

##### 获得所有job和状态

| **/jobs**                     |                         |
| :---------------------------- | ----------------------- |
| `GET`                         | Response code: `200 OK` |
| 返回所有job的概览和当前状态。 |                         |
| **Request**                   |                         |
| **Response**                  |                         |

##### client提交job

| **/jobs**                                                    |                               |
| :----------------------------------------------------------- | ----------------------------- |
| `POST`                                                       | Response code: `202 Accepted` |
| 提交一个job。此调用主要供Flink客户端使用。Submits a job. This call is primarily intended to be used by the Flink client. 该调用需要一个多部份/表单数据请求，请求包含用于序列化JobGraph，jars和分布式缓存artifacts的文件上传，和用于JSON负载的名字为"request"的属性名。 |                               |
| **Request**                                                  |                               |
| {   "type" : "object",   "id" : "urn:jsonschema:org:apache:flink:runtime:rest:messages:job:JobSubmitRequestBody",   "properties" : {     "jobGraphFileName" : {       "type" : "string"     },     "jobJarFileNames" : {       "type" : "array",       "items" : {         "type" : "string"       }     },     "jobArtifactFileNames" : {       "type" : "array",       "items" : {         "type" : "object",         "id" : "urn:jsonschema:org:apache:flink:runtime:rest:messages:job:JobSubmitRequestBody:DistributedCacheFile",         "properties" : {           "entryName" : {             "type" : "string"           },           "fileName" : {             "type" : "string"           }         }       }     }   } } |                               |
| **Response**                                                 |                               |

##### 获得job指标

| **/jobs/metrics**                                            |                         |
| ------------------------------------------------------------ | ----------------------- |
| `GET`                                                        | Response code: `200 OK` |
| 提供对聚合job指标的访问。                                    |                         |
| **Query 参数**                                               |                         |
| `get` (可选):以逗号分隔的字符串值列表，用于选择特定指标。<br>`agg` (可选): 以逗号分隔的应计算的聚合模式列表。可用的聚合有: "min, max, sum, avg"。<br>`jobs` (可选): 以逗号分隔的32个字符的十六进制字符串列表，用于选择特定job。 |                         |
| **Request**                                                  |                         |
| **Response**                                                 |                         |

##### 获得job概览

| **/jobs/overview**  |                         |
| ------------------- | ----------------------- |
| `GET`               | Response code: `200 OK` |
| 返回所有job的概览。 |                         |
| **Request**         |                         |
| **Response**        |                         |

### /jobs/:jobid

##### 获得job详细信息

| **/jobs/:jobid**                                |                         |
| ----------------------------------------------- | ----------------------- |
| `GET`                                           | Response code: `200 OK` |
| 返回job的详细信息。                             |                         |
| **路径参数**                                    |                         |
| `jobid` - 标识job的32个字符的十六进制字符串值。 |                         |
| **Request**                                     |                         |
| **Response**                                    |                         |

##### 停止job

| **/jobs/:jobid**                                             |                               |
| ------------------------------------------------------------ | ----------------------------- |
| `PATCH`                                                      | Response code: `202 Accepted` |
| 停止job。                                                    |                               |
| **路径参数**                                                 |                               |
| `jobid` - 标识job的32个字符的十六进制字符串值。              |                               |
| **Query 参数**                                               |                               |
| `mode` (可选): 指定终止模式的字符串值。支持的值有:"cancel, stop"。 |                               |
| **Request**                                                  |                               |
| **Response**                                                 |                               |

##### 获得job累加器

| **/jobs/:jobid/accumulators**                                |                         |
| ------------------------------------------------------------ | ----------------------- |
| `GET`                                                        | Response code: `200 OK` |
| 返回job所有任务的累加器，这些累加器跨各个子任务聚合。        |                         |
| **路径参数**                                                 |                         |
| `jobid` - 标识job的32个字符的十六进制字符串值。              |                         |
| **Query 参数**                                               |                         |
| `includeSerializedValue` (可选): 布尔值，指定序列化的用户任务累加器是否应该包含在响应中。 |                         |
| **Request**                                                  |                         |
| **Response**                                                 |                         |

##### 获得job checkpoint统计信息

| **/jobs/:jobid/checkpoints**                    |                         |
| ----------------------------------------------- | ----------------------- |
| `GET`                                           | Response code: `200 OK` |
| 返回job checkpoint的统计信息。                  |                         |
| **路径参数**                                    |                         |
| `jobid` - 标识job的32个字符的十六进制字符串值。 |                         |
| **Request**                                     |                         |
| **Response**                                    |                         |

##### 获得checkpoint配置

| **/jobs/:jobid/checkpoints/config**             |                         |
| ----------------------------------------------- | ----------------------- |
| `GET`                                           | Response code: `200 OK` |
| 返回checkpoint配置。                            |                         |
| **路径参数**                                    |                         |
| `jobid` - 标识job的32个字符的十六进制字符串值。 |                         |
| **Request**                                     |                         |
| **Response**                                    |                         |

##### 获得checkpoint detail

| **/jobs/:jobid/checkpoints/details/:checkpointid**           |                         |
| ------------------------------------------------------------ | ----------------------- |
| `GET`                                                        | Response code: `200 OK` |
| 返回某个checkpoint的详细信息。                               |                         |
| **路径参数**                                                 |                         |
| `jobid` - 标识job的32个字符的十六进制字符串值。<br>`checkpointid` - 标识checkpoint的long值。 |                         |
| **Request**                                                  |                         |
| **Response**                                                 |                         |

##### 获得task checkpoint统计信息

| **/jobs/:jobid/checkpoints/details/:checkpointid/subtasks/:vertexid** |                         |
| ------------------------------------------------------------ | ----------------------- |
| `GET`                                                        | Response code: `200 OK` |
| 返回任务及其子任务的checkpoint统计信息。                     |                         |
| **路径参数**                                                 |                         |
| `jobid` - 标识job的32个字符的十六进制字符串值。<br>`checkpointid` - 标识checkpoint的long值。<br><br>`vertexid` - 标识job顶点的32个字符十六进制字符串值。 |                         |
| **Request**                                                  |                         |
| **Response**                                                 |                         |

##### 获得job配置

| **/jobs/:jobid/config**                         |                         |
| ----------------------------------------------- | ----------------------- |
| `GET`                                           | Response code: `200 OK` |
| 返回job配置。                                   |                         |
| **路径参数**                                    |                         |
| `jobid` - 标识job的32个字符的十六进制字符串值。 |                         |
| Request                                         |                         |
| Response                                        |                         |

##### 获得job异常

| **/jobs/:jobid/exceptions**                                  |                         |
| ------------------------------------------------------------ | ----------------------- |
| `GET`                                                        | Response code: `200 OK` |
| 返回job观测到的不可恢复异常。截断flag标识了是否有其他发生了但没列出来的异常，这样做是为了避免响应太臃肿。 |                         |
| **路径参数**                                                 |                         |
| `jobid` - 标识job的32个字符的十六进制字符串值。              |                         |
| **Request**                                                  |                         |
| **Response**                                                 |                         |

##### 获得job执行结果

| **/jobs/:jobid/execution-result**                            |                         |
| ------------------------------------------------------------ | ----------------------- |
| `GET`                                                        | Response code: `200 OK` |
| 返回job执行的结果。允许访问job的执行事件和所有由该job创建的累加器。 |                         |
| **路径参数**                                                 |                         |
| `jobid` - 标识job的32个字符的十六进制字符串值。              |                         |
| **Request**                                                  |                         |
| **Response**                                                 |                         |

##### 访问job指标

| **/jobs/:jobid/metrics**                                   |                         |
| ---------------------------------------------------------- | ----------------------- |
| `GET`                                                      | Response code: `200 OK` |
| 提供对job指标的访问。                                      |                         |
| **路径参数**                                               |                         |
| `jobid` - 标识job的32个字符的十六进制字符串值。            |                         |
| **Query 参数**                                             |                         |
| `get` (可选): 以逗号分隔的字符串值列表，用于选择特定指标。 |                         |
| **Request**                                                |                         |
| **Response**                                               |                         |

##### 获得job的plan

| **/jobs/:jobid/plan**                           |                         |
| ----------------------------------------------- | ----------------------- |
| `GET`                                           | Response code: `200 OK` |
| 返回job的数据流plan。                           |                         |
| **路径参数**                                    |                         |
| `jobid` - 标识job的32个字符的十六进制字符串值。 |                         |
| **Request**                                     |                         |
| **Response**                                    |                         |

##### 触发job rescaling

| **/jobs/:jobid/rescaling**                                   |                         |
| ------------------------------------------------------------ | ----------------------- |
| `PATCH`                                                      | Response code: `200 OK` |
| 触发job的rescaling。此异步操作将为进一步的query标识符返回一个'triggerid'。 |                         |
| **路径参数**                                                 |                         |
| `jobid` - 标识job的32个字符的十六进制字符串值。              |                         |
| **Query 参数**                                               |                         |
| `parallelism` (必填): (可选): 正整数值，指定想要的job并行度。 |                         |
| **Request**                                                  |                         |
| **Response**                                                 |                         |

##### 获得rescaling状态

| **/jobs/:jobid/rescaling/:triggerid**                        |                         |
| ------------------------------------------------------------ | ----------------------- |
| `GET`                                                        | Response code: `200 OK` |
| 返回rescaling操作的状态。                                    |                         |
| **路径参数**                                                 |                         |
| `jobid` - 标识job的32个字符的十六进制字符串值。<br>`triggerid` - 32字符的十六进制字符串，用来标识异步操作的trigger ID。ID返回后操作被触发。 |                         |
| **Request**                                                  |                         |
| **Response**                                                 |                         |

##### 触发savepoint

| **/jobs/:jobid/savepoints**                                  |                               |
| ------------------------------------------------------------ | ----------------------------- |
| `POST`                                                       | Response code: `202 Accepted` |
| 触发savepoint，随后可选取消job。此异步操作将为进一步的query标识符返回一个'triggerid'。 |                               |
| **路径参数**                                                 |                               |
| `jobid` - 标识job的32个字符的十六进制字符串值。              |                               |
| **Request**                                                  |                               |
| {   "type" : "object",   "id" : "urn:jsonschema:org:apache:flink:runtime:rest:messages:job:savepoints:SavepointTriggerRequestBody",   "properties" : {     "target-directory" : {       "type" : "string"     },     "cancel-job" : {       "type" : "boolean"     }   } } |                               |
| **Response**                                                 |                               |

##### 返回savepoint状态

| **/jobs/:jobid/savepoints/:triggerid**                       |                         |
| ------------------------------------------------------------ | ----------------------- |
| `GET`                                                        | Response code: `200 OK` |
| 返回savepoint操作的状态。                                    |                         |
| **路径参数**                                                 |                         |
| `jobid` - 标识job的32个字符的十六进制字符串值。<br>`triggerid` - 32字符的十六进制字符串，用来标识异步操作的trigger ID。ID返回后操作被触发。 |                         |
| **Request**                                                  |                         |
| **Response**                                                 |                         |

##### 停止job

| **/jobs/:jobid/stop**                                        |                               |
| ------------------------------------------------------------ | ----------------------------- |
| `POST`                                                       | Response code: `202 Accepted` |
| 停止具有savepoint的job。(可选)也可以在savepoint刷新等待timer触发的状态之前提交MAX_WATERMARK。此异步操作将为进一步的query标识符返回一个'triggerid'。 |                               |
| **路径参数**                                                 |                               |
| `jobid` - 标识job的32个字符的十六进制字符串值。              |                               |
| **Request**                                                  |                               |
| {   "type" : "object",   "id" : "urn:jsonschema:org:apache:flink:runtime:rest:messages:job:savepoints:stop:StopWithSavepointRequestBody",   "properties" : {     "targetDirectory" : {       "type" : "string"     },     "drain" : {       "type" : "boolean"     }   } } |                               |
| **Response**                                                 |                               |

#### /jobs/:jobid/vertices

##### 获得task detail

| **/jobs/:jobid/vertices/:vertexid**                          |                         |
| ------------------------------------------------------------ | ----------------------- |
| `GET`                                                        | Response code: `200 OK` |
| 返回task的详细信息，包括每个子任务的概要。                   |                         |
| **路径参数**                                                 |                         |
| `jobid` - 标识job的32个字符的十六进制字符串值。<br>`vertexid` - 标识job顶点的32个字符十六进制字符串值。 |                         |
| **Request**                                                  |                         |
| **Response**                                                 |                         |

##### 获得task累加器

| **/jobs/:jobid/vertices/:vertexid/accumulators**             |                         |
| ------------------------------------------------------------ | ----------------------- |
| `GET`                                                        | Response code: `200 OK` |
| 返回task中用户自定义的累加器。累加器横跨所有子任务。         |                         |
| **路径参数**                                                 |                         |
| `jobid` - 标识job的32个字符的十六进制字符串值。<br>`vertexid` - 标识job顶点的32个字符十六进制字符串值。 |                         |
| **Request**                                                  |                         |
| **Response**                                                 |                         |

##### 获得背压信息

| **/jobs/:jobid/vertices/:vertexid/backpressure**             |                         |
| ------------------------------------------------------------ | ----------------------- |
| `GET`                                                        | Response code: `200 OK` |
| 返回job的背压信息，必要时可能会初始化背压抽样。back-pressure：避免丢包的能力。 |                         |
| **路径参数**                                                 |                         |
| `jobid` - 标识job的32个字符的十六进制字符串值。<br>`vertexid` - 标识job顶点的32个字符十六进制字符串值。 |                         |
| **Request**                                                  |                         |
| **Response**                                                 |                         |

##### 访问task指标

| **/jobs/:jobid/vertices/:vertexid/metrics**                  |                         |
| ------------------------------------------------------------ | ----------------------- |
| `GET`                                                        | Response code: `200 OK` |
| 提供对task指标的访问。                                       |                         |
| **路径参数**                                                 |                         |
| `jobid` - 标识job的32个字符的十六进制字符串值。<br>`vertexid` - 标识job顶点的32个字符十六进制字符串值。 |                         |
| **Query 参数**                                               |                         |
| `get` (可选): 以逗号分隔的字符串值列表，用于选择特定指标。   |                         |
| **Request**                                                  |                         |
| **Response**                                                 |                         |

##### 获得sub task累加器

| **/jobs/:jobid/vertices/:vertexid/subtasks/accumulators**    |                         |
| ------------------------------------------------------------ | ----------------------- |
| `GET`                                                        | Response code: `200 OK` |
| 返回所有子任务的用户自定义累加器。                           |                         |
| **路径参数**                                                 |                         |
| `jobid` - 标识job的32个字符的十六进制字符串值。<br>`vertexid` - 标识job顶点的32个字符十六进制字符串值。 |                         |
| **Request**                                                  |                         |
| **Response**                                                 |                         |

##### 访问sub task指标

| **/jobs/:jobid/vertices/:vertexid/subtasks/metrics**         |                         |
| ------------------------------------------------------------ | ----------------------- |
| `GET`                                                        | Response code: `200 OK` |
| 提供对聚合子任务指标的访问。Provides access to aggregated subtask metrics. |                         |
| **路径参数**                                                 |                         |
| `jobid` - 标识job的32个字符的十六进制字符串值。<br>`vertexid` - 标识job顶点的32个字符十六进制字符串值。 |                         |
| **Query 参数**                                               |                         |
| `get` (可选): 以逗号分隔的字符串值列表，用于选择特定指标。<br>`agg` (可选): 以逗号分隔的应计算的聚合模式列表。可用的聚合有: "min, max, sum, avg"。<br/>`subtasks` (可选): 以逗号分隔的整数范围列表， (e.g. "1,3,5-9") 用于选择特定子任务。 |                         |
| **Request**                                                  |                         |
| **Response**                                                 |                         |

##### 获得sub task index

| **/jobs/:jobid/vertices/:vertexid/subtasks/:subtaskindex**   |                         |
| ------------------------------------------------------------ | ----------------------- |
| `GET`                                                        | Response code: `200 OK` |
| 返回子任务当前/最新执行尝试的详细信息。                      |                         |
| **路径参数**                                                 |                         |
| `jobid` - 标识job的32个字符的十六进制字符串值。<br>`vertexid` - 标识job顶点的32个字符十六进制字符串值。<br>`subtaskindex` - 标识子任务的正整数值。 |                         |
| **Request**                                                  |                         |
| **Response**                                                 |                         |

| **/jobs/:jobid/vertices/:vertexid/subtasks/:subtaskindex/attempts/:attempt** |                         |
| ------------------------------------------------------------ | ----------------------- |
| `GET`                                                        | Response code: `200 OK` |
| 返回子任务执行尝试的详细信息。失败/恢复的情况下会发生多次执行尝试。 |                         |
| **路径参数**                                                 |                         |
| `jobid` - 标识job的32个字符的十六进制字符串值。<br>`vertexid` - 标识job顶点的32个字符十六进制字符串值。<br>`subtaskindex` - 标识子任务的正整数值。<br>`attempt` - 标识执行尝试的正整数值。 |                         |
| **Request**                                                  |                         |
| **Response**                                                 |                         |

| **/jobs/:jobid/vertices/:vertexid/subtasks/:subtaskindex/attempts/:attempt/accumulators** |                         |
| ------------------------------------------------------------ | ----------------------- |
| `GET`                                                        | Response code: `200 OK` |
| 返回子任务执行尝试的累加器。失败/恢复的情况下会发生多次执行尝试。 |                         |
| **路径参数**                                                 |                         |
| `jobid` - 标识job的32个字符的十六进制字符串值。<br>`vertexid` - 标识job顶点的32个字符十六进制字符串值。<br>`subtaskindex` - 标识子任务的正整数值。<br>`attempt` - 标识执行尝试的正整数值。 |                         |
| **Request**                                                  |                         |
| **Response**                                                 |                         |

##### 访问sub task指标

| **/jobs/:jobid/vertices/:vertexid/subtasks/:subtaskindex/metrics** |                         |
| ------------------------------------------------------------ | ----------------------- |
| `GET`                                                        | Response code: `200 OK` |
| 提供对子任务指标的访问。                                     |                         |
| **路径参数**                                                 |                         |
| `jobid` - 标识job的32个字符的十六进制字符串值。<br>`vertexid` - 标识job顶点的32个字符十六进制字符串值。`subtaskindex` - 标识子任务的正整数值。 |                         |
| **Query 参数**                                               |                         |
| `get` (可选): 以逗号分隔的字符串值列表，用于选择特定指标。   |                         |
| **Request**                                                  |                         |
| **Response**                                                 |                         |

##### 获得sub task times

| **/jobs/:jobid/vertices/:vertexid/subtasktimes**             |                         |
| ------------------------------------------------------------ | ----------------------- |
| `GET`                                                        | Response code: `200 OK` |
| 返回job的所有子任务的时间相关信息。                          |                         |
| **路径参数**                                                 |                         |
| `jobid` - 标识job的32个字符的十六进制字符串值。<br>`vertexid` - 标识job顶点的32个字符十六进制字符串值。 |                         |
| **Request**                                                  |                         |
| **Response**                                                 |                         |

##### 获得task manger聚合的task info

| **/jobs/:jobid/vertices/:vertexid/taskmanagers**             |                         |
| ------------------------------------------------------------ | ----------------------- |
| `GET`                                                        | Response code: `200 OK` |
| 返回由task manager聚合的任务信息。                           |                         |
| **路径参数**                                                 |                         |
| `jobid` - 标识job的32个字符的十六进制字符串值。<br>`vertexid` - 标识job顶点的32个字符十六进制字符串值。 |                         |
| **Request**                                                  |                         |
| **Response**                                                 |                         |

## /overview

##### 获得Flink集群overview

| **/overview**         |                         |
| --------------------- | ----------------------- |
| `GET`                 | Response code: `200 OK` |
| 返回Flink集群的概述。 |                         |
| **Request**           |                         |
| **Response**          |                         |

## /savepoint-disposal

##### 处置savepoint

| **/savepoint-disposal**                                      |                         |
| ------------------------------------------------------------ | ----------------------- |
| `POST`                                                       | Response code: `200 OK` |
| 触发savepoint的处置。此异步操作将为进一步的query标识符返回一个'triggerid'。 |                         |
| **Request**                                                  |                         |
| {   "type" : "object",   "id" : "urn:jsonschema:org:apache:flink:runtime:rest:messages:job:savepoints:SavepointDisposalRequest",   "properties" : {     "savepoint-path" : {       "type" : "string"     }   } } |                         |
| **Response**                                                 |                         |

| **/savepoint-disposal/:triggerid**                           |                         |
| ------------------------------------------------------------ | ----------------------- |
| `GET`                                                        | Response code: `200 OK` |
| 返回savepoint处置操作的状态。                                |                         |
| **路径参数**                                                 |                         |
| `triggerid` - 32字符的十六进制字符串，用来标识异步操作的trigger ID。ID返回后操作被触发。 |                         |
| **Request**                                                  |                         |
| **Response**                                                 |                         |

## /taskmanagers

##### 获得task manager overview

| **/taskmanagers**            |                         |
| ---------------------------- | ----------------------- |
| `GET`                        | Response code: `200 OK` |
| 返回所有task manager的概览。 |                         |
| **Request**                  |                         |
| **Response**                 |                         |

##### 访问聚合task manager指标

| **/taskmanagers/metrics**                                    |                         |
| ------------------------------------------------------------ | ----------------------- |
| `GET`                                                        | Response code: `200 OK` |
| 提供对聚合task manager指标的访问。                           |                         |
| **Query 参数**                                               |                         |
| `get` (可选): Comma-separated list of string values to select specific metrics.`agg` (可选): 以逗号分隔的应计算的聚合模式列表。可用的聚合有: "min, max, sum, avg"。<br/>`taskmanagers` (可选): 以逗号分隔的32字符十六进制字符串列表，用于选择特定的task manager。 |                         |
| **Request**                                                  |                         |
| **Response**                                                 |                         |

##### 获得task manager detail

| **/taskmanagers/:taskmanagerid**                             |                         |
| ------------------------------------------------------------ | ----------------------- |
| `GET`                                                        | Response code: `200 OK` |
| 返回task manager详细信息。                                   |                         |
| **路径参数**                                                 |                         |
| `taskmanagerid` - 标识task manager的 32 个字符十六进制字符串。 |                         |
| **Request**                                                  |                         |
| **Response**                                                 |                         |

访问task manager指标

| **/taskmanagers/:taskmanagerid/metrics**                     |                         |
| ------------------------------------------------------------ | ----------------------- |
| `GET`                                                        | Response code: `200 OK` |
| 提供对task manager指标的访问。                               |                         |
| **路径参数**                                                 |                         |
| `taskmanagerid` - 标识task manager的 32 个字符十六进制字符串。 |                         |
| **Query 参数**                                               |                         |
| `get` (可选): 用于选择特定指标的字符串值的逗号分隔列表。     |                         |
| **Request**                                                  |                         |
| **Response**                                                 |                         |