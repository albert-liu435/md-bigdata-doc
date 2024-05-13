

### NameServer启动流程

Nameserver的启动类为org.apache.rocketmq.namesrv.NamesrvStartup,在启动之前需要首先配置好rocketmqhome，这个在RocketMQ源码调试中已经介绍过，不再赘述。配置号rocketmqhome之后直接启动NamesrvStartup即可。启动之后进入该方法

```java
    public static NamesrvController main0(String[] args) {

        try {
            //创建NamesrvController并填充NameServConfig和NettyServerConfig的属性值
            NamesrvController controller = createNamesrvController(args);
            //进行Nameserver启动
            start(controller);
            String tip = "The Name Server boot success. serializeType=" + RemotingCommand.getSerializeTypeConfigInThisServer();
            log.info(tip);
            System.out.printf("%s%n", tip);
            return controller;
        } catch (Throwable e) {
            e.printStackTrace();
            System.exit(-1);
        }

        return null;
    }

```

### 1、创建NamesrvController并填充NameServConfig和NettyServerConfig的属性值

```java
public static NamesrvController createNamesrvController(String[] args) throws IOException, JoranException {
        //设置MQ版本信息
        System.setProperty(RemotingCommand.REMOTING_VERSION_KEY, Integer.toString(MQVersion.CURRENT_VERSION));
        //选项配置,利用commons-cli工具类从外部配置信息
        Options options = ServerUtil.buildCommandlineOptions(new Options());
        commandLine = ServerUtil.parseCmdLine("mqnamesrv", args, buildCommandlineOptions(options), new PosixParser());
        if (null == commandLine) {
            System.exit(-1);
            return null;
        }

        //主要为Nameserver的业务参数
        final NamesrvConfig namesrvConfig = new NamesrvConfig();
        //主要为netty网络传输的参数
        final NettyServerConfig nettyServerConfig = new NettyServerConfig();
        //设置监听端口
        nettyServerConfig.setListenPort(9876);
        //NamesrvConfig、NettyServerConfig参数来源有下面两种方式，
        // 1）-c configFile通过-c命令指定配置文件的路径。
        // 2）使用“--属性名 属性值”命令，例如 --listenPort 9876。
        //如果外部传入namesrv.properties配置信息，则进行解析
        if (commandLine.hasOption('c')) {
            //获取namesrv.properties文件
            String file = commandLine.getOptionValue('c');
            if (file != null) {
                InputStream in = new BufferedInputStream(new FileInputStream(file));
                properties = new Properties();
                properties.load(in);
                //将properties配置信息kv设置到namesrvConfig和nettyServerConfig
                MixAll.properties2Object(properties, namesrvConfig);
                MixAll.properties2Object(properties, nettyServerConfig);

                namesrvConfig.setConfigStorePath(file);

                System.out.printf("load config properties file OK, %s%n", file);
                in.close();
            }
        }

        //如果有参数p,则表示打印出所有的参数信息
        if (commandLine.hasOption('p')) {
            InternalLogger console = InternalLoggerFactory.getLogger(LoggerName.NAMESRV_CONSOLE_NAME);
            MixAll.printObjectProperties(console, namesrvConfig);
            MixAll.printObjectProperties(console, nettyServerConfig);
            System.exit(0);
        }
        //解析commandLine参数
        MixAll.properties2Object(ServerUtil.commandLine2Properties(commandLine), namesrvConfig);
        //没有设置rocketmqHome
        if (null == namesrvConfig.getRocketmqHome()) {
            System.out.printf("Please set the %s variable in your environment to match the location of the RocketMQ installation%n", MixAll.ROCKETMQ_HOME_ENV);
            System.exit(-2);
        }
        //日志初始化，默认加载logback_namesrv.xml
        LoggerContext lc = (LoggerContext) LoggerFactory.getILoggerFactory();
        JoranConfigurator configurator = new JoranConfigurator();
        configurator.setContext(lc);
        lc.reset();
        configurator.doConfigure(namesrvConfig.getRocketmqHome() + "/conf/logback_namesrv.xml");

        log = InternalLoggerFactory.getLogger(LoggerName.NAMESRV_LOGGER_NAME);
        //打印相关信息
        MixAll.printObjectProperties(log, namesrvConfig);
        MixAll.printObjectProperties(log, nettyServerConfig);
        //NameServer模块的核心控制器类
        final NamesrvController controller = new NamesrvController(namesrvConfig, nettyServerConfig);

        // remember all configs to prevent discard
        controller.getConfiguration().registerConfig(properties);

        return controller;
    }
```

### 2、进行NameServer启动

```java
    public static NamesrvController start(final NamesrvController controller) throws Exception {

        if (null == controller) {
            throw new IllegalArgumentException("NamesrvController is null");
        }
        //NamesrvController进行初始化
        boolean initResult = controller.initialize();
        if (!initResult) {
            controller.shutdown();
            System.exit(-3);
        }
        //注册JVM钩子函数并启动服务器，以便监听Broker、消息 生产者的网络请求
        Runtime.getRuntime().addShutdownHook(new ShutdownHookThread(log, new Callable<Void>() {
            //如果代码中使用了线程池，一种优雅停机的方式就是注册一个JVM钩子函数，在JVM进程关闭之前，先将线程池关闭，及时释放资源。
            @Override
            public Void call() throws Exception {
                controller.shutdown();
                return null;
            }
        }));
        //NamesrvController启动服务
        controller.start();

        return controller;
    }
```

#### 2.1、NamesrvController进行初始化

```java
public boolean initialize() {
        //加载KV配置，即kvConfig.json
        this.kvConfigManager.load();
        //用来提供请求服务
        this.remotingServer = new NettyRemotingServer(this.nettyServerConfig, this.brokerHousekeepingService);

        //创建一个线程容量为 serverWorkerThreads 的固定长度的线程池，该线程池供 DefaultRequestProcessor 类使用，实现具体的默认的请求命令处理。
        this.remotingExecutor =
                Executors.newFixedThreadPool(nettyServerConfig.getServerWorkerThreads(), new ThreadFactoryImpl("RemotingExecutorThread_"));
        //将DefaultRequestProcessor与创建的线程池remotingExecutor绑定在一起
        this.registerProcessor();

        //NameServer会每隔10s扫描一次brokerLiveTable状态表，如果
        //BrokerLive的lastUpdate-Timestamp时间戳距当前时间超过120s，则认为Broker失效，移除该Broker，关闭与Broker的连接，同时更新
        //topicQueueTable、brokerAddrTable、brokerLiveTable、filterServerTable
        this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

            @Override
            public void run() {
                NamesrvController.this.routeInfoManager.scanNotActiveBroker();
            }
        }, 5, 10, TimeUnit.SECONDS);

        //NameServer每隔10min打印一次KV配置。
        this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

            @Override
            public void run() {
                NamesrvController.this.kvConfigManager.printAllPeriodically();
            }
        }, 1, 10, TimeUnit.MINUTES);


        //配置了tls证书后对tls证书配置文件的监听，配置tls会使通信更加安全
        if (TlsSystemConfig.tlsMode != TlsMode.DISABLED) {
            //注册监听器用来加载SslContext,当配置了tsl证书时
            // Register a listener to reload SslContext
            try {
                fileWatchService = new FileWatchService(
                        new String[]{
                                TlsSystemConfig.tlsServerCertPath,
                                TlsSystemConfig.tlsServerKeyPath,
                                TlsSystemConfig.tlsServerTrustCertPath
                        },
                        new FileWatchService.Listener() {
                            boolean certChanged, keyChanged = false;

                            @Override
                            public void onChanged(String path) {
                                if (path.equals(TlsSystemConfig.tlsServerTrustCertPath)) {
                                    log.info("The trust certificate changed, reload the ssl context");
                                    reloadServerSslContext();
                                }
                                if (path.equals(TlsSystemConfig.tlsServerCertPath)) {
                                    certChanged = true;
                                }
                                if (path.equals(TlsSystemConfig.tlsServerKeyPath)) {
                                    keyChanged = true;
                                }
                                if (certChanged && keyChanged) {
                                    log.info("The certificate and private key changed, reload the ssl context");
                                    certChanged = keyChanged = false;
                                    reloadServerSslContext();
                                }
                            }

                            private void reloadServerSslContext() {
                                ((NettyRemotingServer) remotingServer).loadSslContext();
                            }
                        });
            } catch (Exception e) {
                log.warn("FileWatchService created error, can't load the certificate dynamically");
            }
        }

        return true;
    }
```

NamesrvController初始化主要做了如下的工作
1、加载KV配置，这个暂时没有什么用处
2、创建一个NettyRemotingServer用来提供请求服务
3、创建一个定时任务，默认每隔10s扫描一次brokerLiveTable状态表，如果BrokerLive的lastUpdate-Timestamp时间戳距当前时间超过120s，则认为Broker失效，移除该Broker，关闭与Broker的连接，同时更新topicQueueTable、brokerAddrTable、brokerLiveTable、filterServerTable
4、每隔10min打印一次KV配置
5、如果配置了tls证书，则注册监听器每500ms对tls文件进行检查一下，如果出现变则重新进行加载

##### 2.1.1路由删除

路由的剔除方法在RouteInfoManager类中的scanNotActiveBroker()方法

```java
    /**
     * NameServer每隔10s扫描一次Broker，移除处于未激活状态的Broker。
     */
    public void scanNotActiveBroker() {
        Iterator<Entry<String, BrokerLiveInfo>> it = this.brokerLiveTable.entrySet().iterator();
        //遍历brokerLiveTable路由表
        while (it.hasNext()) {
            Entry<String, BrokerLiveInfo> next = it.next();
            //最后更新时间
            long last = next.getValue().getLastUpdateTimestamp();
            //超过时间120s之后就进行关闭连接，则认为该Broker已不可用，然后将它移除并关闭连接，最后删除与该Broker相关的路由信息
            if ((last + BROKER_CHANNEL_EXPIRED_TIME) < System.currentTimeMillis()) {
                RemotingUtil.closeChannel(next.getValue().getChannel());
                it.remove();
                log.warn("The broker channel expired, {} {}ms", next.getKey(), BROKER_CHANNEL_EXPIRED_TIME);
                this.onChannelDestroy(next.getKey(), next.getValue().getChannel());
            }
        }
    }

    public void onChannelDestroy(String remoteAddr, Channel channel) {
        String brokerAddrFound = null;
        if (channel != null) {
            try {
                try {
                    //申请读锁
                    this.lock.readLock().lockInterruptibly();

                    Iterator<Entry<String, BrokerLiveInfo>> itBrokerLiveTable =
                            this.brokerLiveTable.entrySet().iterator();
                    while (itBrokerLiveTable.hasNext()) {
                        Entry<String, BrokerLiveInfo> entry = itBrokerLiveTable.next();
                        if (entry.getValue().getChannel() == channel) {
                            brokerAddrFound = entry.getKey();
                            break;
                        }
                    }
                } finally {
                    this.lock.readLock().unlock();
                }
            } catch (Exception e) {
                log.error("onChannelDestroy Exception", e);
            }
        }

        if (null == brokerAddrFound) {
            brokerAddrFound = remoteAddr;
        } else {
            log.info("the broker's channel destroyed, {}, clean it's data structure at once", brokerAddrFound);
        }

        //根据brokerAddress从brokerLiveTable、filterServerTable中移除Broker相关的信息
        if (brokerAddrFound != null && brokerAddrFound.length() > 0) {

            try {
                try {
                    //申请写锁
                    this.lock.writeLock().lockInterruptibly();
                    //根据brokerAddress从brokerLiveTable、filterServerTable中移除Broker相关的信息
                    this.brokerLiveTable.remove(brokerAddrFound);
                    this.filterServerTable.remove(brokerAddrFound);

                    String brokerNameFound = null;
                    boolean removeBrokerName = false;
                    //遍历brokerAddrTable，从BrokerData中的brokerAddrs中找到 具体的Broker,从BrokerData中将其移除。如果移除后在BrokerData中不再包含其他Broker，则在brokerAddrTable中移除该brokerName对应的条目
                    Iterator<Entry<String, BrokerData>> itBrokerAddrTable =
                            this.brokerAddrTable.entrySet().iterator();
                    while (itBrokerAddrTable.hasNext() && (null == brokerNameFound)) {
                        BrokerData brokerData = itBrokerAddrTable.next().getValue();

                        Iterator<Entry<Long, String>> it = brokerData.getBrokerAddrs().entrySet().iterator();
                        while (it.hasNext()) {
                            Entry<Long, String> entry = it.next();
                            Long brokerId = entry.getKey();
                            String brokerAddr = entry.getValue();

                            if (brokerAddr.equals(brokerAddrFound)) {
                                brokerNameFound = brokerData.getBrokerName();
                                it.remove();
                                log.info("remove brokerAddr[{}, {}] from brokerAddrTable, because channel destroyed",
                                        brokerId, brokerAddr);
                                break;
                            }
                        }

                        if (brokerData.getBrokerAddrs().isEmpty()) {
                            removeBrokerName = true;
                            itBrokerAddrTable.remove();
                            log.info("remove brokerName[{}] from brokerAddrTable, because channel destroyed",
                                    brokerData.getBrokerName());
                        }
                    }
                    //根据BrokerName，从clusterAddrTable中找到Broker并将其从集群中移除。如果移除后，集群中不包含任何Broker，则将该集群从clusterAddrTable中移除
                    if (brokerNameFound != null && removeBrokerName) {
                        Iterator<Entry<String, Set<String>>> it = this.clusterAddrTable.entrySet().iterator();
                        while (it.hasNext()) {
                            Entry<String, Set<String>> entry = it.next();
                            String clusterName = entry.getKey();
                            Set<String> brokerNames = entry.getValue();
                            boolean removed = brokerNames.remove(brokerNameFound);
                            if (removed) {
                                log.info("remove brokerName[{}], clusterName[{}] from clusterAddrTable, because channel destroyed",
                                        brokerNameFound, clusterName);

                                if (brokerNames.isEmpty()) {
                                    log.info("remove the clusterName[{}] from clusterAddrTable, because channel destroyed and no broker in this cluster",
                                            clusterName);
                                    it.remove();
                                }

                                break;
                            }
                        }
                    }
                    //根据BrokerName，遍历所有主题的队列，如果队列中包含当前Broker的队列，则移除，如果topic只包含待移除Broker的队列，从路由表中删除该topic
                    if (removeBrokerName) {
                        Iterator<Entry<String, List<QueueData>>> itTopicQueueTable =
                                this.topicQueueTable.entrySet().iterator();
                        while (itTopicQueueTable.hasNext()) {
                            Entry<String, List<QueueData>> entry = itTopicQueueTable.next();
                            String topic = entry.getKey();
                            List<QueueData> queueDataList = entry.getValue();

                            Iterator<QueueData> itQueueData = queueDataList.iterator();
                            while (itQueueData.hasNext()) {
                                QueueData queueData = itQueueData.next();
                                if (queueData.getBrokerName().equals(brokerNameFound)) {
                                    itQueueData.remove();
                                    log.info("remove topic[{} {}], from topicQueueTable, because channel destroyed",
                                            topic, queueData);
                                }
                            }

                            if (queueDataList.isEmpty()) {
                                itTopicQueueTable.remove();
                                log.info("remove topic[{}] all queue, from topicQueueTable, because channel destroyed",
                                        topic);
                            }
                        }
                    }
                } finally {
                    this.lock.writeLock().unlock();
                }
            } catch (Exception e) {
                log.error("onChannelDestroy Exception", e);
            }
        }
    }

```

RouteInfoManager管理的路由信息如下

![1645085962](.\pic\1645085962.png)

#### 2.2 NamesrvController启动服务

```java
    public void start() throws Exception {
        //NameServer通信服务启动
        this.remotingServer.start();
        //配置文件监听服务启动
        if (this.fileWatchService != null) {
            this.fileWatchService.start();
        }
    }

```

至此 NameServer完成启动，可以作为一个路由中心对外提供服务了