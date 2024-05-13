## Spring Cloud EurekaServer初始化流程

### Eureka初始化配置

在搭建eureka的注册中心的时候，只是需要在应用启动类中加上EnableEurekaServer注解，并在配置文件中配置相应的参数，然后启动应用，注册中心就已经搭建完成了，那么具体的流程是怎么一回事，让我们一探究竟。

EnableEurekaServer注解用于激活Eureka Server的相关配置，我们继续查看EurekaServerMarkerConfiguration，可以发现里面只是在spring容器中创建了一个Maker的实例，正是因为有Marker实例的存在，才会触发EurekaServerAutoConfiguration的自动化配置，完成一系列的Eureka Server的启动

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(EurekaServerMarkerConfiguration.class)
public @interface EnableEurekaServer {

}

@Configuration(proxyBeanMethods = false)
public class EurekaServerMarkerConfiguration {

	@Bean
	public Marker eurekaServerMarkerBean() {
		return new Marker();
	}

	class Marker {

	}

}

```

EurekaServerAutoConfiguration

```java
@Configuration(proxyBeanMethods = false)
@Import(EurekaServerInitializerConfiguration.class)
//当Makers实例存在是，该自动化注解才会生效
@ConditionalOnBean(EurekaServerMarkerConfiguration.Marker.class)
@EnableConfigurationProperties({ EurekaDashboardProperties.class, InstanceRegistryProperties.class })
@PropertySource("classpath:/eureka/server.properties")
public class EurekaServerAutoConfiguration implements WebMvcConfigurer {

	/**
	 * List of packages containing Jersey resources required by the Eureka server.
	 */
	private static final String[] EUREKA_PACKAGES = new String[] { "com.netflix.discovery", "com.netflix.eureka" };

	@Autowired
	private ApplicationInfoManager applicationInfoManager;

	@Autowired
	private EurekaServerConfig eurekaServerConfig;

	@Autowired
	private EurekaClientConfig eurekaClientConfig;

	@Autowired
	private EurekaClient eurekaClient;

	@Autowired
	private InstanceRegistryProperties instanceRegistryProperties;

	/**
	 * A {@link CloudJacksonJson} instance.
	 */
	public static final CloudJacksonJson JACKSON_JSON = new CloudJacksonJson();

	@Bean
	public HasFeatures eurekaServerFeature() {
		return HasFeatures.namedFeature("Eureka Server", EurekaServerAutoConfiguration.class);
	}

    //实例化EurekaController   当我们在浏览器中访问http://localhost:8761/,实际调用的就是EurekaController
	@Bean
	@ConditionalOnProperty(prefix = "eureka.dashboard", name = "enabled", matchIfMissing = true)
	public EurekaController eurekaController() {
		return new EurekaController(this.applicationInfoManager);
	}

	static {
		CodecWrappers.registerWrapper(JACKSON_JSON);
		EurekaJacksonCodec.setInstance(JACKSON_JSON.getCodec());
	}

	@Bean
	public ServerCodecs serverCodecs() {
		return new CloudServerCodecs(this.eurekaServerConfig);
	}

	private static CodecWrapper getFullJson(EurekaServerConfig serverConfig) {
		CodecWrapper codec = CodecWrappers.getCodec(serverConfig.getJsonCodecName());
		return codec == null ? CodecWrappers.getCodec(JACKSON_JSON.codecName()) : codec;
	}

	private static CodecWrapper getFullXml(EurekaServerConfig serverConfig) {
		CodecWrapper codec = CodecWrappers.getCodec(serverConfig.getXmlCodecName());
		return codec == null ? CodecWrappers.getCodec(CodecWrappers.XStreamXml.class) : codec;
	}

	@Bean
	@ConditionalOnMissingBean
	public ReplicationClientAdditionalFilters replicationClientAdditionalFilters() {
		return new ReplicationClientAdditionalFilters(Collections.emptySet());
	}

    //EurekaServer 集群中节点之间同步微服务实例注册表信息的组件,收到客户端的消息后，第一步：先更新本地注册信息；第二步：遍历所有的 PeerEurekaNode，转发给其它节点
	@Bean
	public PeerAwareInstanceRegistry peerAwareInstanceRegistry(ServerCodecs serverCodecs) {
		this.eurekaClient.getApplications(); // force initialization
		return new InstanceRegistry(this.eurekaServerConfig, this.eurekaClientConfig, serverCodecs, this.eurekaClient,
				this.instanceRegistryProperties.getExpectedNumberOfClientsSendingRenews(),
				this.instanceRegistryProperties.getDefaultOpenForTrafficCount());
	}

    //服务器列表管理，PeerEurekaNodes 管理了所有的 PeerEurekaNode 节点,即PeerEurekaNodes是微服务实例节点的集合，一个PeerEurekaNode就代表一个微服务实例
	@Bean
	@ConditionalOnMissingBean
	public PeerEurekaNodes peerEurekaNodes(PeerAwareInstanceRegistry registry, ServerCodecs serverCodecs,
			ReplicationClientAdditionalFilters replicationClientAdditionalFilters) {
		return new RefreshablePeerEurekaNodes(registry, this.eurekaServerConfig, this.eurekaClientConfig, serverCodecs,
				this.applicationInfoManager, replicationClientAdditionalFilters);
	}

    //本地EurekaServer服务的上下文
	@Bean
	@ConditionalOnMissingBean
	public EurekaServerContext eurekaServerContext(ServerCodecs serverCodecs, PeerAwareInstanceRegistry registry,
			PeerEurekaNodes peerEurekaNodes) {
		return new DefaultEurekaServerContext(this.eurekaServerConfig, serverCodecs, registry, peerEurekaNodes,
				this.applicationInfoManager);
	}

    //用于启动EurekaServer服务的实例
	@Bean
	public EurekaServerBootstrap eurekaServerBootstrap(PeerAwareInstanceRegistry registry,
			EurekaServerContext serverContext) {
		return new EurekaServerBootstrap(this.applicationInfoManager, this.eurekaClientConfig, this.eurekaServerConfig,
				registry, serverContext);
	}

	/**
	是一个过滤器
	 * Register the Jersey filter.
	 * @param eurekaJerseyApp an {@link Application} for the filter to be registered
	 * @return a jersey {@link FilterRegistrationBean}
	 */
	@Bean
	public FilterRegistrationBean<?> jerseyFilterRegistration(javax.ws.rs.core.Application eurekaJerseyApp) {
		FilterRegistrationBean<Filter> bean = new FilterRegistrationBean<Filter>();
		bean.setFilter(new ServletContainer(eurekaJerseyApp));
		bean.setOrder(Ordered.LOWEST_PRECEDENCE);
		bean.setUrlPatterns(Collections.singletonList(EurekaConstants.DEFAULT_PREFIX + "/*"));

		return bean;
	}

	/**
	 * Construct a Jersey {@link javax.ws.rs.core.Application} with all the resources
	 * required by the Eureka server.
	 * @param environment an {@link Environment} instance to retrieve classpath resources
	 * @param resourceLoader a {@link ResourceLoader} instance to get classloader from
	 * @return created {@link Application} object
	 */
	@Bean
	public javax.ws.rs.core.Application jerseyApplication(Environment environment, ResourceLoader resourceLoader) {

		ClassPathScanningCandidateComponentProvider provider = new ClassPathScanningCandidateComponentProvider(false,
				environment);

		// Filter to include only classes that have a particular annotation.
		//
		provider.addIncludeFilter(new AnnotationTypeFilter(Path.class));
		provider.addIncludeFilter(new AnnotationTypeFilter(Provider.class));

		// Find classes in Eureka packages (or subpackages)
		//
		Set<Class<?>> classes = new HashSet<>();
		for (String basePackage : EUREKA_PACKAGES) {
			Set<BeanDefinition> beans = provider.findCandidateComponents(basePackage);
			for (BeanDefinition bd : beans) {
				Class<?> cls = ClassUtils.resolveClassName(bd.getBeanClassName(), resourceLoader.getClassLoader());
				classes.add(cls);
			}
		}

		// Construct the Jersey ResourceConfig
		Map<String, Object> propsAndFeatures = new HashMap<>();
		propsAndFeatures.put(
				// Skip static content used by the webapp
				ServletContainer.PROPERTY_WEB_PAGE_CONTENT_REGEX,
				EurekaConstants.DEFAULT_PREFIX + "/(fonts|images|css|js)/.*");

		DefaultResourceConfig rc = new DefaultResourceConfig(classes);
		rc.setPropertiesAndFeatures(propsAndFeatures);

		return rc;
	}

	@Bean
	@ConditionalOnBean(name = "httpTraceFilter")
	public FilterRegistrationBean<?> traceFilterRegistration(@Qualifier("httpTraceFilter") Filter filter) {
		FilterRegistrationBean<Filter> bean = new FilterRegistrationBean<Filter>();
		bean.setFilter(filter);
		bean.setOrder(Ordered.LOWEST_PRECEDENCE - 10);
		return bean;
	}

	@Configuration(proxyBeanMethods = false)
	protected static class EurekaServerConfigBeanConfiguration {

		@Bean
		@ConditionalOnMissingBean
		public EurekaServerConfig eurekaServerConfig(EurekaClientConfig clientConfig) {
			EurekaServerConfigBean server = new EurekaServerConfigBean();
			if (clientConfig.shouldRegisterWithEureka()) {
				// Set a sensible default if we are supposed to replicate
				server.setRegistrySyncRetries(5);
			}
			return server;
		}

	}

	/**
	 * {@link PeerEurekaNodes} which updates peers when /refresh is invoked. Peers are
	 * updated only if <code>eureka.client.use-dns-for-fetching-service-urls</code> is
	 * <code>false</code> and one of following properties have changed.
	 * <p>
	 * </p>
	 * <ul>
	 * <li><code>eureka.client.availability-zones</code></li>
	 * <li><code>eureka.client.region</code></li>
	 * <li><code>eureka.client.service-url.&lt;zone&gt;</code></li>
	 * </ul>
	 */
	static class RefreshablePeerEurekaNodes extends PeerEurekaNodes
			implements ApplicationListener<EnvironmentChangeEvent> {

		private ReplicationClientAdditionalFilters replicationClientAdditionalFilters;

		RefreshablePeerEurekaNodes(final PeerAwareInstanceRegistry registry, final EurekaServerConfig serverConfig,
				final EurekaClientConfig clientConfig, final ServerCodecs serverCodecs,
				final ApplicationInfoManager applicationInfoManager,
				final ReplicationClientAdditionalFilters replicationClientAdditionalFilters) {
			super(registry, serverConfig, clientConfig, serverCodecs, applicationInfoManager);
			this.replicationClientAdditionalFilters = replicationClientAdditionalFilters;
		}

		@Override
		protected PeerEurekaNode createPeerEurekaNode(String peerEurekaNodeUrl) {
			JerseyReplicationClient replicationClient = JerseyReplicationClient.createReplicationClient(serverConfig,
					serverCodecs, peerEurekaNodeUrl);

			this.replicationClientAdditionalFilters.getFilters().forEach(replicationClient::addReplicationClientFilter);

			String targetHost = hostFromUrl(peerEurekaNodeUrl);
			if (targetHost == null) {
				targetHost = "host";
			}
			return new PeerEurekaNode(registry, targetHost, peerEurekaNodeUrl, replicationClient, serverConfig);
		}

		@Override
		public void onApplicationEvent(final EnvironmentChangeEvent event) {
			if (shouldUpdate(event.getKeys())) {
				updatePeerEurekaNodes(resolvePeerUrls());
			}
		}

		/*
		 * Check whether specific properties have changed.
		 */
		protected boolean shouldUpdate(final Set<String> changedKeys) {
			assert changedKeys != null;

			// if eureka.client.use-dns-for-fetching-service-urls is true, then
			// service-url will not be fetched from environment.
			if (this.clientConfig.shouldUseDnsForFetchingServiceUrls()) {
				return false;
			}

			if (changedKeys.contains("eureka.client.region")) {
				return true;
			}

			for (final String key : changedKeys) {
				// property keys are not expected to be null.
				if (key.startsWith("eureka.client.service-url.")
						|| key.startsWith("eureka.client.availability-zones.")) {
					return true;
				}
			}
			return false;
		}

	}

	class CloudServerCodecs extends DefaultServerCodecs {

		CloudServerCodecs(EurekaServerConfig serverConfig) {
			super(getFullJson(serverConfig), CodecWrappers.getCodec(CodecWrappers.JacksonJsonMini.class),
					getFullXml(serverConfig), CodecWrappers.getCodec(CodecWrappers.JacksonXmlMini.class));
		}

	}

}

```

### EurekaServerInitializerConfiguration完成Eureka初始化

EurekaServerInitializerConfiguration配置类实现了SmartLifecycle，所以在完成Ioc完成所有Bean初始化后，来执行实现了SmartLifecycle类的start()方法，即EurekaServerInitializerConfiguration中的start()方法

```java
@Override
public void start() {
   new Thread(() -> {
      try {
         // TODO: is this class even needed now?
          //启动Eureka 服务
         eurekaServerBootstrap.contextInitialized(EurekaServerInitializerConfiguration.this.servletContext);
         log.info("Started Eureka Server");
		//发布EurekaRegistryAvailableEvent事件
         publish(new EurekaRegistryAvailableEvent(getEurekaServerConfig()));
          //设置运行状态为true
         EurekaServerInitializerConfiguration.this.running = true;
          //发布EurekaServerStartedEvent事件
         publish(new EurekaServerStartedEvent(getEurekaServerConfig()));
      }
      catch (Exception ex) {
         // Help!
         log.error("Could not initialize Eureka servlet context", ex);
      }
   }).start();
}
```

进入到 eurekaServerBootstrap.contextInitialized(ServletContext context)的方法中，该方法主要进行了初始化Eureka的环境信息和初始化Eureka的应用上下文信息。

```java
public void contextInitialized(ServletContext context) {
   try {
   //初始化环境信息，该方法是一空方法
      initEurekaEnvironment();
      //初始化Eureka应用上下文信息
      initEurekaServerContext();

      context.setAttribute(EurekaServerContext.class.getName(), this.serverContext);
   }
   catch (Throwable e) {
      log.error("Cannot bootstrap eureka server :", e);
      throw new RuntimeException("Cannot bootstrap eureka server :", e);
   }
}

	protected void initEurekaServerContext() throws Exception {
		// For backward compatibility
		JsonXStream.getInstance().registerConverter(new V1AwareInstanceInfoConverter(), XStream.PRIORITY_VERY_HIGH);
		XmlXStream.getInstance().registerConverter(new V1AwareInstanceInfoConverter(), XStream.PRIORITY_VERY_HIGH);

		if (isAws(this.applicationInfoManager.getInfo())) {
			this.awsBinder = new AwsBinderDelegate(this.eurekaServerConfig, this.eurekaClientConfig, this.registry,
					this.applicationInfoManager);
			this.awsBinder.start();
		}

		EurekaServerContextHolder.initialize(this.serverContext);

		log.info("Initialized server context");

		// Copy registry from neighboring eureka node
        //遍历eurekaClient.getApplications获取服务信息，并将服务信息注册到自己的registry中。
		int registryCount = this.registry.syncUp();
        //1、修改服务的状态为UP
        //2、调用父类的postInit 开启一个剔除定时任务，每隔一定时间(默认60秒)执行一次，从当前服务清单中把超时（默认90秒）没有续约剔
		this.registry.openForTraffic(this.applicationInfoManager, registryCount);

		// Register all monitoring statistics.
        //注册所有的监控统计信息
		EurekaMonitors.registerAllStats();
	}
```

### 总结

SpringCloud Eureka Server作为注册中心，主要完成下面的三件事情

1、启动后，从其他节点拉取服务注册信息

2、运行过程中，启动一个evict任务，定时剔除没有按时Renew的服务

3、接受到其他节点的Rgister,Renew,Cancel请求信息后，会将这些信息同步至其他注册中心的节点。

























