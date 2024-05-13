## HDFS

### HDFS[结构](https://github.com/Dr11ft/BigDataGuide/tree/master/Hadoop)

#### HDFS组成架构

![HDFS组成架构](.\pic\HDFS组成架构.png)

这种架构主要由四个部分组成，分别为HDFS Client、NameNode、DataNode和Secondary NameNode。下面我们分别介绍这四个组成部分。
1）Client：就是客户端。
  （1）文件切分。文件上传HDFS的时候，Client将文件切分成一个一个的Block，然后进行存储；
  （2）与NameNode交互，获取文件的位置信息；
  （3）与DataNode交互，读取或者写入数据；
  （4）Client提供一些命令来管理HDFS，比如启动或者关闭HDFS；
  （5）Client可以通过一些命令来访问HDFS；
2）NameNode：就是Master，它是一个主管、管理者。
  （1）管理HDFS的名称空间；
  （2）管理数据块（Block）映射信息；
  （3）配置副本策略；
  （4）处理客户端读写请求。
3）DataNode：就是Slave。NameNode下达命令，DataNode执行实际的操作。
  （1）存储实际的数据块；
  （2）执行数据块的读/写操作。
4）Secondary NameNode：并非NameNode的热备。当NameNode挂掉的时候，它并不能马上替换NameNode并提供服务。
  （1）辅助NameNode，分担其工作量；
  （2）定期合并Fsimage和Edits，并推送给NameNode；
  （3）在紧急情况下，可辅助恢复NameNode。

#### HDFS[架构的稳定性](https://github.com/heibaiying/BigData-Notes/blob/master/notes/Hadoop-HDFS.md)

##### 心跳机制和重新复制

每个 DataNode 定期向 NameNode 发送心跳消息，如果超过指定时间没有收到心跳消息，则将 DataNode 标记为死亡。NameNode 不会将任何新的 IO 请求转发给标记为死亡的 DataNode，也不会再使用这些 DataNode 上的数据。 由于数据不再可用，可能会导致某些块的复制因子小于其指定值，NameNode 会跟踪这些块，并在必要的时候进行重新复制。

##### 数据完整性

由于存储设备故障等原因，存储在 DataNode 上的数据块也会发生损坏。为了避免读取到已经损坏的数据而导致错误，HDFS 提供了数据完整性校验机制来保证数据的完整性，具体操作如下：

当客户端创建 HDFS 文件时，它会计算文件的每个块的 `校验和`，并将 `校验和` 存储在同一 HDFS 命名空间下的单独的隐藏文件中。当客户端检索文件内容时，它会验证从每个 DataNode 接收的数据是否与存储在关联校验和文件中的 `校验和` 匹配。如果匹配失败，则证明数据已经损坏，此时客户端会选择从其他 DataNode 获取该块的其他可用副本。

##### 元数据的磁盘故障

`FsImage` 和 `EditLog` 是 HDFS 的核心数据，这些数据的意外丢失可能会导致整个 HDFS 服务不可用。为了避免这个问题，可以配置 NameNode 使其支持 `FsImage` 和 `EditLog` 多副本同步，这样 `FsImage` 或 `EditLog` 的任何改变都会引起每个副本 `FsImage` 和 `EditLog` 的同步更新。

#### HDFS数据流

##### HDFS的写数据流程

![HDFS的写数据流程](.\pic\HDFS的写数据流程.png)

步骤：
  1）客户端通过Distributed FileSystem模块向NameNode请求上传文件，NameNode检查目标文件是否已存在，父目录是否存在。
  2）NameNode返回是否可以上传。
  3）客户端请求第一个 block上传到哪几个datanode服务器上。
  4）NameNode返回3个datanode节点，分别为dn1、dn2、dn3。
  5）客户端通过FSDataOutputStream模块请求dn1上传数据，dn1收到请求会继续调用dn2，然后dn2调用dn3，将这个通信管道建立完成。
  6）dn1、dn2、dn3逐级应答客户端。
  7）客户端开始往dn1上传第一个block（先从磁盘读取数据放到一个本地内存缓存），以packet为单位，dn1收到一个packet就会传给dn2，dn2传给dn3；dn1每传一个packet会放入一个应答队列等待应答。
  8）当一个block传输完成之后，客户端再次请求NameNode上传第二个block的服务器。（重复执行3-7步）。

##### HDFS的读数据流程

![HDFS的读数据流程](.\pic\HDFS的读数据流程.png)

步骤：
  1）客户端通过Distributed FileSystem向NameNode请求下载文件，NameNode通过查询元数据，找到文件块所在的DataNode地址。
  2）挑选一台DataNode（就近原则，然后随机）服务器，请求读取数据。
  3）DataNode开始传输数据给客户端（从磁盘里面读取数据输入流，以packet为单位来做校验）。
  4）客户端以packet为单位接收，先在本地缓存，然后写入目标文件。

#### 机架感知

（2）低版本Hadoop副本节点选择
  第一个副本在Client所处的节点上。如果客户端在集群外，随机选一个。
  第二个副本和第一个副本位于不相同机架的随机节点上。
  第三个副本和第二个副本位于相同机架，节点随机。

![机架感知1](.\pic\机架感知1.png)

（3）Hadoop2.7.2副本节点选择
  第一个副本在Client所处的节点上。如果客户端在集群外，随机选一个。
  第二个副本和第一个副本位于相同机架，随机节点。
  第三个副本位于不同机架，随机节点。

![机架感知2](.\pic\机架感知2.png)



## MapReduce

[Hadoop——分布式计算框架MapReduce](MapReduce.md)

[Hadoop——MapReduce案例](MapReduce案例.md)





