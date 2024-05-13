## SpringCloud Eureka中的@LoadBalanced注解

在前面的示例代码中，当我们搭建spring cloud Eureka的server及消费者和生产者集群时，当我们访问http://localhost:10001/consumer/hello的时候，会交替的返回hello euureka port:9091;hello euureka port:9092;hello euureka port:9093;,其中这里其主要作用的就是@LoadBalanced注解。下面我们来一看究竟。

如我们在消费者端只需要添加一段如下代码，就可以实现负载均衡。

```java
    @Bean
    @LoadBalanced
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
```

### LoadBalancerAutoConfiguration

要想LoadBalanced注解生效，我们首先需要看一下LoadBalancerAutoConfiguration类中的自动化配置

```java
@Configuration(proxyBeanMethods = false)
//类路径下必须存在RestTemplate才会生效
@ConditionalOnClass(RestTemplate.class)
//Spring容器内必须存在这个接口的Bean才会生效，默认为BlockingLoadBalancerClient实例
@ConditionalOnBean(LoadBalancerClient.class)
@EnableConfigurationProperties(LoadBalancerProperties.class)
public class LoadBalancerAutoConfiguration {
	//获取容器中的所有标注为LoadBalanced的bean
	@LoadBalanced
	@Autowired(required = false)
	private List<RestTemplate> restTemplates = Collections.emptyList();

	@Autowired(required = false)
	private List<LoadBalancerRequestTransformer> transformers = Collections.emptyList();
	
    //这里主要为RestTemplate添加拦截器,
	@Bean
	public SmartInitializingSingleton loadBalancedRestTemplateInitializerDeprecated(
			final ObjectProvider<List<RestTemplateCustomizer>> restTemplateCustomizers) {
		return () -> restTemplateCustomizers.ifAvailable(customizers -> {
			for (RestTemplate restTemplate : LoadBalancerAutoConfiguration.this.restTemplates) {
				for (RestTemplateCustomizer customizer : customizers) {
					customizer.customize(restTemplate);
				}
			}
		});
	}

    //实例化LoadBalancerRequestFactory，里面包含了创建LoadBalancerRequest的createRequest方法
	@Bean
	@ConditionalOnMissingBean
	public LoadBalancerRequestFactory loadBalancerRequestFactory(LoadBalancerClient loadBalancerClient) {
		return new LoadBalancerRequestFactory(loadBalancerClient, this.transformers);
	}

	@Configuration(proxyBeanMethods = false)
	@Conditional(RetryMissingOrDisabledCondition.class)
	static class LoadBalancerInterceptorConfig {

        //使用LoadBalancerClient和LoadBalancerRequestFactory实例化LoadBalancerInterceptor，负载均衡拦截器，最终将拦截器会添加到
        //RestTemplate
		@Bean
		public LoadBalancerInterceptor loadBalancerInterceptor(LoadBalancerClient loadBalancerClient,
				LoadBalancerRequestFactory requestFactory) {
			return new LoadBalancerInterceptor(loadBalancerClient, requestFactory);
		}

        //向容器内放入一个RestTemplateCustomizer 定制器
        //给所有的RestTemplate添加拦截器LoadBalancerInterceptor,使其具备负载均衡的能力
		@Bean
		@ConditionalOnMissingBean
		public RestTemplateCustomizer restTemplateCustomizer(final LoadBalancerInterceptor loadBalancerInterceptor) {
			return restTemplate -> {
				List<ClientHttpRequestInterceptor> list = new ArrayList<>(restTemplate.getInterceptors());
				list.add(loadBalancerInterceptor);
				restTemplate.setInterceptors(list);
			};
		}

	}

	private static class RetryMissingOrDisabledCondition extends AnyNestedCondition {

		RetryMissingOrDisabledCondition() {
			super(ConfigurationPhase.REGISTER_BEAN);
		}

		@ConditionalOnMissingClass("org.springframework.retry.support.RetryTemplate")
		static class RetryTemplateMissing {

		}

		@ConditionalOnProperty(value = "spring.cloud.loadbalancer.retry.enabled", havingValue = "false")
		static class RetryDisabled {

		}

	}

	/**
	 * Auto configuration for retry mechanism.
	 */
	@Configuration(proxyBeanMethods = false)
	@ConditionalOnClass(RetryTemplate.class)
	public static class RetryAutoConfiguration {

		@Bean
		@ConditionalOnMissingBean
		public LoadBalancedRetryFactory loadBalancedRetryFactory() {
			return new LoadBalancedRetryFactory() {
			};
		}

	}

}

```

@ConditionalOnClass：某个class位于类路径上，才会实例化一个Bean,代码示例中只有在RestTemplate存在的情况下，才会进行自动化部署。

@ConditionalOnBean：仅仅在当前上下文中存在某个对象时，才会实例化一个Bean,代码示例中只有在LoadBalancerClient对象存在的情况下，才会进行自动化部署，其中LoadBalancerClient的默认实现类为BlockingLoadBalancerClient

经过前面的自动化配置，此时RestTemplate因为有了拦截器的加持，已经具备了负载均衡的能力。即所用通过RestTemplate发送的请求都会通过LoadBalancerInterceptor拦截器。

### LoadBalancerInterceptor

找到LoadBalancerInterceptor源码后，我们发现拦截器在获取请求的originalUri 和serviceName 后，最终委托给LoadBalancerClient去执行，其中一个serviceName有可能对应很多个实际的服务，所以LoadBalancerClient可以运用相关的均衡算法找出符合条件的服务做最终的请求。部分源码如下：

```java
	@Override
	public ClientHttpResponse intercept(final HttpRequest request, final byte[] body,
			final ClientHttpRequestExecution execution) throws IOException {
		final URI originalUri = request.getURI();
		String serviceName = originalUri.getHost();
		Assert.state(serviceName != null, "Request URI does not contain a valid hostname: " + originalUri);
		return this.loadBalancer.execute(serviceName, this.requestFactory.createRequest(request, body, execution));
	}
```

### BlockingLoadBalancerClient

我们看一下BlockingLoadBalancerClient的具体继承关系。

![f2d8bdf945f132699835bbf7b356deb](.\pic\f2d8bdf945f132699835bbf7b356deb.png)

ServiceInstanceChooser是一个接口，定义了选择相应的服务实例的方法

```
public interface ServiceInstanceChooser {

	//根据调用者传入的serviceId，通过负载均衡算法选择一个具体的服务实例来提供 服务
	ServiceInstance choose(String serviceId);

	//根据调用者传入的serviceId和请求，通过负载均衡算法选择一个具体的服务实例来提供 服务
	<T> ServiceInstance choose(String serviceId, Request<T> request);

}

```

LoadBalancerClient定义了执行请求的方法

```
public interface LoadBalancerClient extends ServiceInstanceChooser {

	//执行请求
	<T> T execute(String serviceId, LoadBalancerRequest<T> request) throws IOException;
	<T> T execute(String serviceId, ServiceInstance serviceInstance, LoadBalancerRequest<T> request) throws IOException;
	//重新构造url,即将原来的替换为实际服务的地址url
	URI reconstructURI(ServiceInstance instance, URI original);

}
```

我们看一下默认的实现类BlockingLoadBalancerClient,真正执行负载均衡的实现类

```java
@SuppressWarnings({ "unchecked", "rawtypes" })
public class BlockingLoadBalancerClient implements LoadBalancerClient {

	private final LoadBalancerClientFactory loadBalancerClientFactory;

	private final LoadBalancerProperties properties;

	public BlockingLoadBalancerClient(LoadBalancerClientFactory loadBalancerClientFactory,
			LoadBalancerProperties properties) {
		this.loadBalancerClientFactory = loadBalancerClientFactory;
		this.properties = properties;

	}

//执行请求
	@Override
	public <T> T execute(String serviceId, LoadBalancerRequest<T> request) throws IOException {
		//hit是一种算法，这里可以认为是 分组，在指定的组里面进行负载均衡
		String hint = getHint(serviceId);
		LoadBalancerRequestAdapter<T, DefaultRequestContext> lbRequest = new LoadBalancerRequestAdapter<>(request,
				new DefaultRequestContext(request, hint));
		Set<LoadBalancerLifecycle> supportedLifecycleProcessors = getSupportedLifecycleProcessors(serviceId);
		supportedLifecycleProcessors.forEach(lifecycle -> lifecycle.onStart(lbRequest));
		//这里会选择真正的提供服务的实例
		ServiceInstance serviceInstance = choose(serviceId, lbRequest);
		if (serviceInstance == null) {
			supportedLifecycleProcessors.forEach(lifecycle -> lifecycle.onComplete(
					new CompletionContext<>(CompletionContext.Status.DISCARD, lbRequest, new EmptyResponse())));
			throw new IllegalStateException("No instances available for " + serviceId);
		}
		//调用真正的服务实例，执行请求
		return execute(serviceId, serviceInstance, lbRequest);
	}

	@Override
	public <T> T execute(String serviceId, ServiceInstance serviceInstance, LoadBalancerRequest<T> request)
			throws IOException {
		DefaultResponse defaultResponse = new DefaultResponse(serviceInstance);
		Set<LoadBalancerLifecycle> supportedLifecycleProcessors = getSupportedLifecycleProcessors(serviceId);
		Request lbRequest = request instanceof Request ? (Request) request : new DefaultRequest<>();
		supportedLifecycleProcessors
				.forEach(lifecycle -> lifecycle.onStartRequest(lbRequest, new DefaultResponse(serviceInstance)));
		try {
			//执行真正的请求，并返回请求的结果
			T response = request.apply(serviceInstance);
			Object clientResponse = getClientResponse(response);
			supportedLifecycleProcessors
					.forEach(lifecycle -> lifecycle.onComplete(new CompletionContext<>(CompletionContext.Status.SUCCESS,
							lbRequest, defaultResponse, clientResponse)));
			return response;
		}
		catch (IOException iOException) {
			supportedLifecycleProcessors.forEach(lifecycle -> lifecycle.onComplete(
					new CompletionContext<>(CompletionContext.Status.FAILED, iOException, lbRequest, defaultResponse)));
			throw iOException;
		}
		catch (Exception exception) {
			supportedLifecycleProcessors.forEach(lifecycle -> lifecycle.onComplete(
					new CompletionContext<>(CompletionContext.Status.FAILED, exception, lbRequest, defaultResponse)));
			ReflectionUtils.rethrowRuntimeException(exception);
		}
		return null;
	}

	private <T> Object getClientResponse(T response) {
		ClientHttpResponse clientHttpResponse = null;
		if (response instanceof ClientHttpResponse) {
			clientHttpResponse = (ClientHttpResponse) response;
		}
		if (clientHttpResponse != null) {
			try {
				return new ResponseData(clientHttpResponse, null);
			}
			catch (IOException ignored) {
			}
		}
		return response;
	}

	private Set<LoadBalancerLifecycle> getSupportedLifecycleProcessors(String serviceId) {
		return LoadBalancerLifecycleValidator.getSupportedLifecycleProcessors(
				loadBalancerClientFactory.getInstances(serviceId, LoadBalancerLifecycle.class),
				DefaultRequestContext.class, Object.class, ServiceInstance.class);
	}

	@Override
	public URI reconstructURI(ServiceInstance serviceInstance, URI original) {
		return LoadBalancerUriTools.reconstructURI(serviceInstance, original);
	}

	@Override
	public ServiceInstance choose(String serviceId) {
		return choose(serviceId, REQUEST);
	}

	@Override
	public <T> ServiceInstance choose(String serviceId, Request<T> request) {
		//查找负载均衡器，默认采用的是RoundRobinLoadBalancer,即简单轮询负载均衡器，
		ReactiveLoadBalancer<ServiceInstance> loadBalancer = loadBalancerClientFactory.getInstance(serviceId);
		if (loadBalancer == null) {
			return null;
		}

		Response<ServiceInstance> loadBalancerResponse = Mono.from(loadBalancer.choose(request)).block();
		if (loadBalancerResponse == null) {
			return null;
		}
		//获取真正的提供服务的实例
		return loadBalancerResponse.getServer();
	}

	private String getHint(String serviceId) {
		String defaultHint = properties.getHint().getOrDefault("default", "default");
		String hintPropertyValue = properties.getHint().get(serviceId);
		return hintPropertyValue != null ? hintPropertyValue : defaultHint;
	}

}

```

上面就是负载均衡的策略，我们也可以按照自己的方式实现自己的负载均衡

### 自定义负载均衡的实现

我们在原有的rookie-springcloud-eureka-consumer进行代码的修改，我们首先定义自己的负载均衡注解MyLoadBalanced

```java
@Target({ ElementType.FIELD, ElementType.PARAMETER, ElementType.METHOD })
@Retention(RetentionPolicy.RUNTIME)
@Qualifier
public @interface MyLoadBalanced {
}

```

自定义请求的拦截器

```java
public class MyHttpInterceptor implements ClientHttpRequestInterceptor {
    @Override
    public ClientHttpResponse intercept(HttpRequest request, byte[] body, ClientHttpRequestExecution execution) throws IOException {

        MyHttpRequest newRequest = new MyHttpRequest(request);

        return execution.execute(newRequest, body);
    }
}

```

自定义请求MyHttpRequest

```java
public class MyHttpRequest implements HttpRequest {



    String [] uriStr=new String[]{"http://127.0.0.1:9091/producer/hello","http://127.0.0.1:9092/producer/hello","http://127.0.0.1:9093/producer/hello"};



    private HttpRequest httpRequest;


    public MyHttpRequest(HttpRequest httpRequest){
        this.httpRequest=httpRequest;
    }

    @Override
    public String getMethodValue() {
        return this.httpRequest.getMethodValue();
    }

    @Override
    public URI getURI() {
        int max=3,min=0;
        int ran2 = (int) (Math.random()*(max-min)+min);


        try {
            URI newUri = new URI(uriStr[ran2]);
            return newUri;
        } catch (Exception e) {
            e.printStackTrace();
        }

        return this.httpRequest.getURI();
    }

    @Override
    public HttpHeaders getHeaders() {
        return this.httpRequest.getHeaders();
    }
}
```

注入到spring容器中

```
@Configuration
public class MyLoadBalancerAutoConfiguration {


    @Autowired(required = false)
    @MyLoadBalanced
    private List<RestTemplate> myTemplates = Collections.emptyList();


    @Bean
    public SmartInitializingSingleton myLoadBalancedRestTemplateInitializer() {
        return new SmartInitializingSingleton() {

            public void afterSingletonsInstantiated() {
                for (RestTemplate tpl : myTemplates) {
                    MyHttpInterceptor mi = new MyHttpInterceptor();
                    List list = new ArrayList(tpl.getInterceptors());
                    list.add(mi);
                    tpl.setInterceptors(list);
                }
            }
        };
    }


}
```

应用启动类

```java
@EnableDiscoveryClient
@SpringBootApplication
public class RookieEurekaConsumerServerApplication {


//    @Bean
//    @LoadBalanced
//    public RestTemplate restTemplate() {
//        return new RestTemplate();
//    }


    //自定义拦截器注解
    @Bean
    @MyLoadBalanced
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }


    public static void main(String[] args) {
        SpringApplication.run(RookieEurekaConsumerServerApplication.class, args);
    }

}
```

这样我们就实现了自己的负载均衡策略了，负载策略按照随机策略获取真正的实例进行响应。我们在访问http://localhost:10001/consumer/hello的时候，会随机返回hello euureka port:9091;hello euureka port:9092;hello euureka port:9093;中的一个。
