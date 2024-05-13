## Eureka架构原理

### eureka架构图

图片为Eureka github官方给出的高可用架构图，[github](https://github.com/Netflix/eureka/wiki/Eureka-at-a-glance)

![eureka_architecture](.\pic\eureka_architecture.png)

其中us-east-1c,us-east-1d,us-east-1e分别代表不同的数据中心即机房。Eureka Server为注册中心，实际就是我们demo中的rookie-springcloud-eureka-server集群，Application Service是服务提供者，可以认为是我们demo中的rookie-springcloud-eureka-producer集群实例。Application Client是服务的 消费者，可以认为是我们demo中的rookie-springcloud-eureka-consumer实例。

#### Eureka Server(注册中心)

服务的注册中心主要完成三件大事

1、启动后，从其他节点拉取服务注册信息

2、运行过程中，启动一个evict任务，定时剔除没有按时Renew的服务

3、接受到其他节点的Rgister,Renew,Cancel请求信息后，会将这些信息同步至其他注册中心的节点。

#### Register(服务注册)

服务的提供者，服务的消费者以及注册中心(一般集群情况下)都会通过Rest的请求方式注册到注册中心，包括服务的ip,服务名称，url等信息，注册中心收到这些信息后，会将这些信息保存registry中。

#### Replicate(服务同步)

在集群情况下，Eureka Server服务之间是相互注册的，不同的服务Eureka Server实例之间会进行信息的同步，用来保证信息的一致性

#### Renew(服务续约)

在服务注册后，Eureka Client会每隔一定时间(默认30s)发送一次心跳来通知Eureka Server(服务注册中心),说明该Eureka Client运行正常，一直处于可用的 状态，防止Eureka Server剔除。默认情况下，Eureka Server每隔一定时间(90s)内没有收到Eureka Client的心跳，Server端将会从注册列表中删除该实例。

#### Cancel(服务下线)

服务正常停止时会向注册中心发送下线通知，Eureka Server(服务注册中心)在收到请求后，就会把该服务状态置为下线（DOWN），并把该下线事件传播出去。

#### evict(服务剔除)

Eureka Client 因为网络故障等原因导致不能提供服务,此时Eureka Server(服务注册中心)不再收到Eureka Client客户端的心跳超过一定时间时(默认90s)，此时Eureka Server(服务注册中心)会判定该实例已经挂掉，不能再提供服务，会将该服务从服务清单中剔除。

Get Registy(获取注册表信息)













