### kafka的设计是什么样的

1、kafka将消息以topic为单位进行归纳
2、向kafka topic发布消息的程序称为prodcers
3、将预定topics并消费消息的程序称为consumer
4、kafka以集群的方式运行，可以由一个或多个服务组成，每个服务叫做一个broker
5、producers通过网络将消息发送到kafka集群，集群向消费者提供消息

### kafka数据传输事务的定义有那三种

1、最多一次:消息不会被重复发送，最多被传输一次,但也有可能一次不传输
2、最少传输一次：消息不会被遗漏发送，最少被传输一次，但也有可能被重复传输
3、精确一次：不会漏传输也不会重复传输，每个消息都传输一次而且仅仅被传输一次，这是大家所期望的

### kafka判断一个节点是否与存活的两个条件

1、节点必须可以维护和zookeeper的连接，zookeeper通过心跳机制检查每个节点的连接
2、如果节点是个follower,他必须能及时的同步leader的写操作，延时不能太久

### producer是否直接将数据发送到broker的leader(主节点)

producer直接将数据发送到broker的leader(主节点),不需要在多个节点进行分发，为了帮助producer做到这点，所有的kafka节点都可以及时的告知；那些节点时活动的目标，目标topic的leader在哪。这样producer就可以直接将消息发送到目的地啦
kafka是否可以消费指定分区的消息
kafka消费消息时，向broker发出"fetch"请求去消费特定分区的消息，customer指定消息在日志中的偏移量(offset),就可以消费从这个位置开始的消息，customer拥有offset的控制，可以向后回滚去重新消费之前的消息，这很有意义的。

### kafka consumer是否可以消费指定分区消息？

kafka consumer消费消息时,向broker发出“fetch"请求去消费待定分区的消息，consumer指定消息在日志中的偏移量（offset)，就可以消费从这个位置开始的消息,customer拥有了offset的控制权，可以向后回滚去重新消费之前的消息，这是有意义的
kafka消息采用的是pull模式还是push模式？
kafka遵循了一种大部分消息系统共同的传统的设计：producer将消息推送到broker，consumer从broker拉取消息到下游consumer.
好处：consumer自动拉取可以减轻压力，另一个就是consumer可以自主决定是否批量从broker拉取数据
缺点：consumer需要在循环中轮询，直到消息到达，为了避免这一点，kafka有个参数可以让consumer阻塞直到消息到达。

### kafka存储在硬盘上的消息格式是什么？

消息由一个固定长度的头部和可变长度的字节数组组成。头部包含了一个版本号和CRC32校验码。
1、消息长度: 4bytes(value:1+4+n)
2、版本号:1byte
3、CRC校验码：4 bytes
4、具体消息：n bytes

### kafka高效文件存储区设计特点

1、kafka把topic中的一个parition大文件分成多个小文件段，通过多个小文件段就容易定期清除或删除已经消费完文件，减少磁盘占用

