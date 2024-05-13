

# HDFS

## [HDFS](https://www.cnblogs.com/jimmy888/p/13562676.html)

#### HDFS写数据流程

![hdfs写数据流程示意图v_shizhan03](.\pic\hdfs写数据流程示意图v_shizhan03.png)

1、根namenode通信请求上传文件，namenode检查目标文件是否已存在，父目录是否存在
2、namenode返回是否可以上传
3、client请求第一个 block该传输到哪些datanode服务器上
4、namenode返回3个datanode服务器ABC
5、client请求3台dn中的一台A上传数据（本质上是一个RPC调用，建立pipeline），A收到请求会继续调用B，然后B调用C，将真个pipeline建立完成，逐级返回客户端
6、client开始往A上传第一个block（先从磁盘读取数据放到一个本地内存缓存），以packet为单位，A收到一个packet就会传给B，B传给C；A每传一个packet会放入一个应答队列等待应答
7、当一个block传输完成之后，client再次请求namenode上传第二个block的服务器。

#### HDFS读数据流程

![hdfs读数据流程示意图v_shizhan03](.\pic\hdfs读数据流程示意图v_shizhan03.png)

1、跟namenode通信查询元数据，找到文件块所在的datanode服务器
2、挑选一台datanode（就近原则，然后随机）服务器，请求建立socket流
3、datanode开始发送数据（从磁盘里面读取数据放入流，以packet为单位来做校验）
4、客户端以packet为单位接收，现在本地缓存，然后写入目标文件

## NAMENODE

#### NAMENODE的职责

负责客户端请求的响应

元数据的管理（查询，修改）

#### 元数据管理

namenode对数据的管理采用了三种存储形式：

内存元数据(NameSystem)

磁盘元数据镜像文件

数据操作日志文件（可通过日志运算出元数据）

#### 元数据存储机制

A、内存中有一份完整的元数据(**内存meta data**)

B、磁盘有一个“准完整”的元数据镜像（**fsimage**）文件(在namenode的工作目录中)

C、用于衔接内存metadata和持久化元数据镜像fsimage之间的操作日志（**edits****文件**）*注：当客户端对**hdfs**中的文件进行新增或者修改操作，操作记录首先被记入edits**日志文件中，当客户端操作成功后，相应的元数据会更新到内存meta.data**中*

#### 元数据的checkpoint

每隔一段时间，会由secondary namenode将namenode上积累的所有edits和一个最新的fsimage下载到本地，并加载到内存进行merge（这个过程称为checkpoint）

![黑马checkpoint](.\pic\黑马checkpoint.png)

##### checkpoint附带作用

每隔一段时间，会由secondary namenode将namenode上积累的所有edits和一个最新的fsimage下载到本地，并加载到内存进行merge（这个过程称为checkpoint）

## DATANODE

#### DATANODE工作职责

存储管理用户的文件块数据

定期向namenode汇报自身所持有的block信息（通过心跳信息上报）

（这点很重要，因为，当集群中发生某些block副本失效时，集群如何恢复block初始副本数量的问题）

#### DATANODE掉线判断时参数

datanode进程死亡或者网络故障造成datanode无法与namenode通信，namenode不会立即把该节点判定为死亡，要经过一段时间，这段时间暂称作超时时长。HDFS默认的超时时长为10分钟+30秒。如果定义超时时间为timeout，则超时时长的计算公式为：

​    timeout = 2 * heartbeat.recheck.interval + 10 * dfs.heartbeat.interval。

​    而默认的heartbeat.recheck.interval 大小为5分钟，dfs.heartbeat.interval默认为3秒。

​    需要注意的是hdfs-site.xml 配置文件中的heartbeat.recheck.interval的单位为毫秒，dfs.heartbeat.interval的单位为秒。所以，举个例子，如果heartbeat.recheck.interval设置为5000（毫秒），dfs.heartbeat.interval设置为3（秒，默认），则总的超时时间为40秒。

## HDFS安全模式

安全模式是HDFS所处的一种特殊状态，在这种状态下，文件系统**只接受读数据请求，而不接受删除、修改等变更请求**。在NameNode主节点启动时，HDFS首先进入安全模式，DataNode在启动的时候会向namenode汇报可用的block等状态（通过心跳heartbeat向namenode汇报，如果**99%的block的块的副本数>=1**，就可以退出安全模式），当整个系统达到安全标准时，HDFS自动离开安全模式。

**如果HDFS处于安全模式下，则文件block不能进行任何的副本复制操作**，因此达到最小的副本数量要求是基于datanode启动时的状态来判定的，启动时不会再做任何复制（从而达到最小副本数量要求），hdfs集群刚启动的时候，**默认30s**的时间是出于安全期的，只有过了30s之后，集群脱离了安全期，然后才可以对集群进行操作

**通过shell命令进入和退出安全模式：**



# MAPREDUCE

## MAPREDUCE框架结构及核心运行机制

#### 结构

一个完整的mapreduce程序在分布式运行时有三类实例进程：

1、MRAppMaster：负责整个程序的过程调度及状态协调

2、mapTask：负责map阶段的整个数据处理流程

3、ReduceTask：负责reduce阶段的整个数据处理流程

#### MR运行流程

![流程示意图](C:\work\IDEAWorkSpace\rookie-project\haizhilangzigitee\bigdata-doc\hadoop\pic\流程示意图.png)

1、 一个mr程序启动的时候，最先启动的是MRAppMaster，MRAppMaster启动后根据本次job的描述信息，计算出需要的maptask实例数量，然后向集群申请机器启动相应数量的maptask进程

2、 maptask进程启动之后，根据给定的数据切片范围进行数据处理，主体流程为：

a)     利用客户指定的inputformat来获取RecordReader读取数据，形成输入KV对

b)     将输入KV对传递给客户定义的map()方法，做逻辑运算，并将map()方法输出的KV对收集到缓存

c)     将缓存中的KV对按照K分区排序后不断溢写到磁盘文件

3、 MRAppMaster监控到所有maptask进程任务完成之后，会根据客户指定的参数启动相应数量的reducetask进程，并告知reducetask进程要处理的数据范围（数据分区）

4、 Reducetask进程启动之后，根据MRAppMaster告知的待处理数据所在位置，从若干台maptask运行所在机器上获取到若干个maptask输出结果文件，并在本地进行重新归并排序，然后按照相同key的KV为一个组，调用客户定义的reduce()方法进行逻辑运算，并收集运算输出的结果KV，然后调用客户指定的outputformat将结果数据输出到外部存储

#### MapTask并行度

![hadop切片机制](.\pic\hadop切片机制.png)

##### FileInputFormat中默认的切片机制：

a)     简单地按照文件的内容长度进行切片

b)     切片大小，默认等于block大小

c)     切片时不考虑数据集整体，而是逐个针对每一个文件单独切片

#### ReduceTask并行度

reducetask的并行度同样影响整个job的执行并发度和执行效率，但与maptask的并发数由切片数决定不同，Reducetask数量的决定是可以直接手动设置：

//默认值是1，手动设置为4

job.setNumReduceTasks(4);

如果数据分布不均匀，就有可能在reduce阶段产生数据倾斜

#### MR运行全流程

![mapreduce运行全流程](.\pic\mapreduce运行全流程.png)



### MapTask和ReduceTask[工作机制](https://www.cnblogs.com/jimmy888/p/13568659.html)

### MapTask工作机制

![MapTask工作机制2-794212616](.\pic\MapTask工作机制2-794212616.png)

![MapTask工作机制-425311668](.\pic\MapTask工作机制-425311668.png)

**Read阶段--》Map阶段--》Collect阶段--》spill阶段--》Combine阶段**

#### Read阶段

有个文件`hello.txt`大小为`200M`，客户端首先获取待处理文件信息，然后根据参数配置，形成一个任务分配的规划。

再调用`submit()`方法，把要执行的`jar`包、`Job.xml`(`Job`信息)、`Job.split`（分片信息）等提交到hdfs集群。

然后把程序提交到`yarn`集群运行，yarn会生成一个`MrAppMaster`（一个进程），用来控制`maptask`和`reducetask`的启动。因为分片信息已经提交到`hdfs`集群，那`MrAppMaster`就会去获取分片信息，计算出`Maptask`数量。`MapTask`通过用户编写的`RecordReader`，从输入`InputSplit`中解析出一个个`key/value`。

#### Map阶段

该节点主要是将解析出的`key/value`交给用户编写`map()`函数处理，并产生一系列新的`key/value`。

#### Collect收集阶段

在用户编写`map()`函数中，当数据处理完成后，一般会调用`OutputCollector.collect()`输出结果。在该函数内部，它会将生成的`key/value`分区（调用`Partitioner`），并写入一个**环形内存缓冲区**中。环形缓冲区的默认大小是`100M`。

#### spill阶段

当环形缓冲区满`80%`后，就会打开流，`MapReduce`会将数据写到本地文件系统磁盘上，**生成一个临时文件**，然后关闭流。当环形缓冲区再次满`80%`后，又会打开流，开始溢写。因此，有可能生成多个临时文件。

需要注意的是，**将数据写入本地磁盘之前**，先要对数据**进行一次本地排序，并在必要时对数据进行规约、压缩、分区等操作**。

溢写阶段详情：

步骤1：利用快速排序算法对缓存区内的数据进行排序，排序方式是，先按照分区编号`Partition`进行排序，然后按照`key`进行排序。这样，经过排序后，**数据以分区为单位聚集在一起，且同一分区内所有数据按照key有序。**

步骤2：按照分区编号由小到大依次将每个分区中的数据写入任务工作目录下的临时文件`output/spillN.out`（`N`表示当前溢写次数）中。如果用户设置了`Combiner`，则写入文件之前，对每个分区中的数据进行一次聚集操作。

步骤3：将分区数据的元信息写到内存索引数据结构`SpillRecord`中，其中每个分区的元信息包括在临时文件中的偏移量、压缩前数据大小和压缩后数据大小。如果当前内存索引大小超过`1MB`，则将内存索引写到文件`output/spillN.out.index`中。

#### Combine阶段

当所有数据处理完成后，`MapTask`对所有临时文件进行一次合并，以确保最终只会生成一个数据文件。

当所有数据处理完后，`MapTask`会将所有临时文件合并成一个大文件，并保存到文件`output/file.out`中，同时生成相应的索引文件`output/file.out.index`。

在进行文件合并过程中，`MapTask`以分区为单位进行合并。对于某个分区，它将采用**多轮递归合并**的方式。每轮合并`io.sort.factor（`默认10）个文件，并将产生的文件重新加入待合并列表中，对文件排序后，重复以上过程，直到最终得到一个大文件。

**让每个`MapTask`最终只生成一个数据文件，可避免同时打开大量文件和同时读取大量小文件产生的随机读取带来的开销。**

### ReduceTask工作机制

![ReduceTask-1467341324](.\pic\ReduceTask-1467341324.png)

#### Copy阶段

`ReduceTask`从各个`MapTask`上远程拷贝一片数据，并针对某一片数据，如果其大小超过一定阈值，则写到磁盘（`hdfs`文件系统）上，否则直接放到内存中。

#### Merge阶段

在远程拷贝数据的同时，`ReduceTask`启动了两个后台线程**分别对内存和磁盘上的文件进行合并**，以防止内存使用过多或磁盘上文件过多。

#### Sort阶段

按照`MapReduce`语义，用户编写`reduce()`函数的输入数据是按`key`进行聚集的一组数据，如`（hello,Iterable(1,1,1,1))`。

为了将`key`相同的数据聚在一起，`Hadoop`采用了基于排序的策略。由于各个`MapTask`已经实现对自己的处理结果进行了局部排序，因此，**`ReduceTask`只需对所有数据进行一次归并排序即可**。

#### Reduce阶段

`reduce()`函数将计算结果写到`HDFS`上。默认是使用`TextOutputFormat`类来写。

#### MapReduce完整流程

第一步：读取文件，解析成为`key，value`对

第二步：自定义map逻辑接受`k1,v1`，转换成为新的k2,v2输出

第三步：分区`Partition`。将相同`key`的数据发送到同一个`reduce`里面去

第四步：排序，`map`阶段分区内的数据进行排序

第五步：`combiner`。调优过程，对数据进行`map`阶段的合并

第六步：将环形缓冲区的数据进行溢写到本地磁盘小文件

第七步：归并排序，对本地磁盘溢写小文件进行归并排序

第八步：等待`reduceTask`启动线程来进行拉取数据

第九步：`reduceTask`启动线程拉取属于自己分区的数据

第十步：从`mapTask`拉取回来的数据继续进行归并排序

第十一步：进行`groupingComparator`分组操作

第十二步：调用`reduce`逻辑，写出数据

第十三步：通过`outputFormat`进行数据输出，写到文件，一个`reduceTask`对应一个文件



## YARN

#### YARN架构

![yarn架构](.\pic\yarn架构.png)

##### ResourceManager

`ResourceManager` 通常在独立的机器上以后台进程的形式运行，它是整个集群资源的主要协调者和管理者。`ResourceManager` 负责给用户提交的所有应用程序分配资源，它根据应用程序优先级、队列容量、ACLs、数据位置等信息，做出决策，然后以共享的、安全的、多租户的方式制定分配策略，调度集群资源。

##### NodeManager

`NodeManager` 是 YARN 集群中的每个具体节点的管理者。主要负责该节点内所有容器的生命周期的管理，监视资源和跟踪节点健康。具体如下：

启动时向 `ResourceManager` 注册并定时发送心跳消息，等待 `ResourceManager` 的指令；

维护 `Container` 的生命周期，监控 `Container` 的资源使用情况；

管理任务运行时的相关依赖，根据 `ApplicationMaster` 的需要，在启动 `Container` 之前将需要的程序及其依赖拷贝到本地。

##### ApplicationMaster

在用户提交一个应用程序时，YARN 会启动一个轻量级的进程 `ApplicationMaster`。`ApplicationMaster` 负责协调来自 `ResourceManager` 的资源，并通过 `NodeManager` 监视容器内资源的使用情况，同时还负责任务的监控与容错。具体如下：

根据应用的运行状态来决定动态计算资源需求；

向 `ResourceManager` 申请资源，监控申请的资源的使用情况；

跟踪任务状态和进度，报告资源的使用情况和应用的进度信息；

负责任务的容错。

##### Container

`Container` 是 YARN 中的资源抽象，它封装了某个节点上的多维度资源，如内存、CPU、磁盘、网络等。当 AM 向 RM 申请资源时，RM 为 AM 返回的资源是用 `Container` 表示的。YARN 会为每个任务分配一个 `Container`，该任务只能使用该 `Container` 中描述的资源。`ApplicationMaster` 可在 `Container` 内运行任何类型的任务。例如，`MapReduce ApplicationMaster` 请求一个容器来启动 map 或 reduce 任务，而 `Giraph ApplicationMaster` 请求一个容器来运行 Giraph 任务。

##### mapreduce&yarn的工作机制

![mapreduce&yarn的工作机制----吸星大法](.\pic\mapreduce&yarn的工作机制----吸星大法.png)

##### YARN工作原理

![yarn工作机制-1480981544](.\pic\yarn工作机制-1480981544.png)

1. MR程序提交到客户端所在的节点。

2. YarnRunner向ResourceManager申请一个Application。

3. RM将该应用程序的资源路径返回给YarnRunner。

4. 该程序将运行所需资源提交到HDFS上。

5. 程序资源提交完毕后，申请运行mrAppMaster。

6. RM将用户的请求初始化成一个Task。

7. 其中一个NodeManager领取到Task任务。

8. 该NodeManager创建容器Container，并产生MRAppmaster。

9. Container从HDFS上拷贝资源到本地。

10. MRAppmaster向RM 申请运行MapTask资源。

11. RM将运行MapTask任务分配给另外两个NodeManager，另两个NodeManager分别领取任务并创建容器。

12. MR向两个接收到任务的NodeManager发送程序启动脚本，这两个NodeManager分别启动MapTask，MapTask对数据分区排序。

13. MrAppMaster等待所有MapTask运行完毕后，向RM申请容器，运行ReduceTask。

14. ReduceTask向MapTask获取相应分区的数据。

15. 程序运行完毕后，MR会向RM申请注销自己。

##### 详细过程

![yarn详细过程-708994489](.\pic\yarn详细过程-708994489.png)

1. `MR`程序（可以理解成`jar`包）提交到客户端所在的节点，MR程序里面有一个`Job.waitforcompletion()`方法，这个方法会生成一个`jobsummiter`实例对象，然后这个过程还会通过`jobsummiter`调用`runJob()`方法。
2. 客户端与`ResourceManager`进行通信，调用`getApplication()`方法申请一个应用，RM收到申请后，返回分配给客户端一个`applicaion id`。
3. 客户端判断`MR`程序的输入路径是否存在，若不存在则报错。然后客户端会计算分片信息，比如根据文件大小得到要多少个分片。
   \4. 上述操作无误后，客户端把`Job`资源提交到`hdfs`。`Job`资源包括`jar`包、`job`的配置信息（`job.xml`)、分片信息。
4. 客户端`summit`提交 `job`，`RM`收到提交后，根据各个`NodeManager`上运行的资源状况向某个`NM`节点发送启动容器的请求，该`NM`收到`RM`的请求会启动一个容器`Container`。然后在该容器上启动一个`MRAppMaster`进程（进程运行在`JVM`虚拟机里）。
5. `MRAppMaster`进行初始化，生成一些对象，用于记录`task`的完成情况。
6. `MRAppMaster`从`hdfs`获取分片信息（得知`maptask`个数）以及`reducetask`个数。
7. `MRAppMaster`向`RM`申请资源，启动容器，用于运行`task`。`MRAppMaster`会优先为`Maptask`申请资源。`RM`会返回要在哪几个`Nodemanager`开启容器、这个容器有多少资源等信息。
8. `MRAppMaster`与`RM`返回的所有`NM`节点，进行通信。`NM`收到信息后，就会在本地启动容器。
9. 在容器内启用`YarnChild`类，从`hdfs`拉取`job.ja`r包、配置信息等一些运行`task`的资源。然后每个容器就可以启动一个`maptask`了。
10. 接下来`maptask`运行的过程，就跟我们前面的学的`maptask`工作机制一样了。
11. 当整个`job`中的`maptask`的进程达到`5%`的时候，`MRAppMaster`就开始向`RM`为`reducetask`申请资源，启动容器。（每个`Maptask`运行过程都会向`MRAppMaster`上报进度信息，进度信息可看成是已经输入的数据占所有数据的百分比）
12. 然后`RM`再次返回要开启容器的`NM`节点信息，`MRAppMaster`再跟这些`NM`通信，`NM`启动容器，启用`YarnChild`，最后开启`reducetask`。
13. `reducetask`要从完成`100%`进度的`maptask`的所在节点拷贝数据过来自身所在节点。（`maptask`会向`MRAppMaster`上报进度信息，这些信息保存在`MRAppMaster`初始化生成的对象里，`reducetask`会不定期跟`MRAppMaster`通信，询问哪个`maptask`完成了）
14. 接下来`reducetask`的过程就跟我们前面学的`reducetask`工作机制一样了，最终会把输出结果保存到`hdfs`里。

***1\***|***3\*****补充：**

- 每个`task`完成后，容器都会被释放掉。

- 每个`maptask`完成后，`maptask`在本地磁盘产生的临时文件都会被删除掉

- 程序运行完毕后，`MR`会向`RM`申请注销自己，`MRAppMaster`所在容器也会释放掉。

- 我们在运行MR程序的时候可以发现，会打印一些进度、计数器等信息出来，这是因为每个task在运行过程都会向`MRAppMaster`上报，如果在编写MR程序的时候，为`job.waitForCompletion();`传入参数`true`，客户端就会每隔几秒向`MRAppMaster`获取最新进度信息，取到就打印出来。

- `MRAppMaster`为`maptask`申请资源时，会把`maptask`的要使用的数据分片的所在节点信息一带发送给`RM`，如果分片所在的节点的资源是足够使用的话，`RM`会优先考虑这些`NM`节点，这样`maptask`运行直接从本地磁盘读数据就可以了，更高效点，这是**移动计算不移动数据**的思想。

- 企业生产当中，一般是一个节点（机器）同时运行`datanode+namanode`。效率更高点。

- 如果`task`运行失败，`MRAppMaster`会知道，并会为`task`重新申请资源容器，且优先考虑其它的`NM`节点，每个`task`有`4`次重试机会,如果超过`4`次,整个`job`失败。

- `MRAppMaster`会向`RM`发送心跳，如果超过一定时间，`RM`没有收到心跳，会判断`MRAppMaster`挂掉了。如果`MRAppMaster`运行失败，会启动一个新容器，运行`MRAppMaster`。新的`MRAppMaster`可以得知之前的`task`完成情况，因为启用了历史日志服务，任务完成情况会记录在内。

- `maptask`和`reducetask`的进度等信息都会保存在`MRAppMaster`初始化产生的对象里。

- 每个`job`对应一个`MRAppMaster`。

## 高可用架构

HDFS高可用

![高可用](.\pic\高可用.png)

HDFS 高可用架构主要由以下组件所构成：

- **Active NameNode 和 Standby NameNode**：两台 NameNode 形成互备，一台处于 Active 状态，为主 NameNode，另外一台处于 Standby 状态，为备 NameNode，只有主 NameNode 才能对外提供读写服务。

- **主备切换控制器 ZKFailoverController**：ZKFailoverController 作为独立的进程运行，对 NameNode 的主备切换进行总体控制。ZKFailoverController 能及时检测到 NameNode 的健康状况，在主 NameNode 故障时借助 Zookeeper 实现自动的主备选举和切换，当然 NameNode 目前也支持不依赖于 Zookeeper 的手动主备切换。

- **Zookeeper 集群**：为主备切换控制器提供主备选举支持。

- **共享存储系统**：共享存储系统是实现 NameNode 的高可用最为关键的部分，共享存储系统保存了 NameNode 在运行过程中所产生的 HDFS 的元数据。主 NameNode 和 NameNode 通过共享存储系统实现元数据同步。在进行主备切换的时候，新的主 NameNode 在确认元数据完全同步之后才能继续对外提供服务。

- **DataNode 节点**：除了通过共享存储系统共享 HDFS 的元数据信息之外，主 NameNode 和备 NameNode 还需要共享 HDFS 的数据块和 DataNode 之间的映射关系。DataNode 会同时向主 NameNode 和备 NameNode 上报数据块的位置信息。

#### 基于 QJM 的共享存储系统的数据同步机制分析

目前 Hadoop 支持使用 Quorum Journal Manager (QJM) 或 Network File System (NFS) 作为共享的存储系统，这里以 QJM 集群为例进行说明：Active NameNode 首先把 EditLog 提交到 JournalNode 集群，然后 Standby NameNode 再从 JournalNode 集群定时同步 EditLog，当 Active NameNode 宕机后， Standby NameNode 在确认元数据完全同步之后就可以对外提供服务。

需要说明的是向 JournalNode 集群写入 EditLog 是遵循 “过半写入则成功” 的策略，所以你至少要有 3 个 JournalNode 节点，当然你也可以继续增加节点数量，但是应该保证节点总数是奇数。同时如果有 2N+1 台 JournalNode，那么根据过半写的原则，最多可以容忍有 N 台 JournalNode 节点挂掉。

![JournalNode](.\pic\JournalNode.png)

#### NameNode主备切换

![namenode-主备切换](.\pic\namenode-主备切换.png)

1. HealthMonitor 初始化完成之后会启动内部的线程来定时调用对应 NameNode 的 HAServiceProtocol RPC 接口的方法，对 NameNode 的健康状态进行检测。 2. HealthMonitor 如果检测到 NameNode 的健康状态发生变化，会回调 ZKFailoverController 注册的相应方法进行处理。 3. 如果 ZKFailoverController 判断需要进行主备切换，会首先使用 ActiveStandbyElector 来进行自动的主备选举。 4. ActiveStandbyElector 与 Zookeeper 进行交互完成自动的主备选举。 5. ActiveStandbyElector 在主备选举完成后，会回调 ZKFailoverController 的相应方法来通知当前的 NameNode 成为主 NameNode 或备 NameNode。 6. ZKFailoverController 调用对应 NameNode 的 HAServiceProtocol RPC 接口的方法将 NameNode 转换为 Active 状态或 Standby 状态。

####   YARN高可用

YARN ResourceManager 的高可用与 HDFS NameNode 的高可用类似，但是 ResourceManager 不像 NameNode ，没有那么多的元数据信息需要维护，所以它的状态信息可以直接写到 Zookeeper 上，并依赖 Zookeeper 来进行主备选举。

![yarn高可用27](.\pic\yarn高可用27.png)





