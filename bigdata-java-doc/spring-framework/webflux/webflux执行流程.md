

# WebFlux

spring-webflux是Spring5添加的新的模块，用于web开发，功能和spring-web mvc类似,webflux使用当前一种比较流行的响应式编程出现的框架。

传统的web框架，如springmvc是基于Servlet容器，而Webflux是一种异步非阻塞的框架，核心是基于Reactor的相关API实现的。

## 异步非阻塞

异步和同步：异步和同步是针对调用者，调用者发送请求，如果等着对方回应之后才能去做其他事情就是同步，如果发送请求之后不等着对方回应就去做其他事情就是异步。

阻塞与非阻塞：阻塞与非阻塞是针对被调用者，被调用者在收到请求之后，做完请求任务之后才给出反馈就是阻塞，收到请求之后马上给出反馈然后再去做事情就是非阻塞。

## webflux特点

非阻塞式：在有限的资源下，提高系统吞吐量和伸缩性，以Reactor为基础实现响应式编程

函数式编程：Spring5框架基于java8，webflux使用java8函数式编程方式实现路由请求

springmvc和webflux

先看一下官网的一张图

![spring-mvc-and-webflux-venn](.\pic\spring-mvc-and-webflux-venn.png)

两个框架都可以使用注解方式，都可以运行在tomcat、jetty等容器中

Springmvc采用命令式编程，webflux采用异步响应式编程

响应式编程是一种面向数据流和变化传播的编程范式。这意味着可以在编程语言中很方便地表达静态或动态的数据流，而相关的计算模型会自动将变化的值通过数据流进行传播。 电子表格程序就是响应式编程的一个例子。单元格可以包含字面值或类似"=B1+C1"的公式，而包含公式的单元格的值会依据其他单元格的值的变化而变化。

## webflux执行流程

webflux的执行流程和springmvc的执行流程相似，都是由一个控制器进行分发，只不过springmvc用的是DispatcherServlet，而webflux用的是DispatcherHandler。

webflux的三大组件

HandlerMapping：请求查询到处理的方法
HandlerAdapter：真正负责请求处理
HandlerResultHandler：响应结果处理

我们首先看一下DispatcherHandler的中的初始化方法

```java
protected void initStrategies(ApplicationContext context) {
   //获取HandlerMapping及其子类型的bean,HandlerMapping根据请求request获取handler执行链
   Map<String, HandlerMapping> mappingBeans = BeanFactoryUtils.beansOfTypeIncludingAncestors(
         context, HandlerMapping.class, true, false);
   ArrayList<HandlerMapping> mappings = new ArrayList<>(mappingBeans.values());
   AnnotationAwareOrderComparator.sort(mappings);
   this.handlerMappings = Collections.unmodifiableList(mappings);
	//获取HandlerAdapter及其子类型的bean
   Map<String, HandlerAdapter> adapterBeans = BeanFactoryUtils.beansOfTypeIncludingAncestors(
         context, HandlerAdapter.class, true, false);
	
   this.handlerAdapters = new ArrayList<>(adapterBeans.values());
   AnnotationAwareOrderComparator.sort(this.handlerAdapters);
//获取HandlerResultHandler及其子类型的bean
   Map<String, HandlerResultHandler> beans = BeanFactoryUtils.beansOfTypeIncludingAncestors(
         context, HandlerResultHandler.class, true, false);

   this.resultHandlers = new ArrayList<>(beans.values());
   AnnotationAwareOrderComparator.sort(this.resultHandlers);
}
```

处理请求的方法

```java
@Override
public Mono<Void> handle(ServerWebExchange exchange) {
    //如果handlerMappings为null,表示资源没有找到，直接返回404
   if (this.handlerMappings == null) {
      return createNotFoundError();
   }
   if (CorsUtils.isPreFlightRequest(exchange.getRequest())) {
      return handlePreFlight(exchange);
   }
   return Flux.fromIterable(this.handlerMappings)
       //根据请求地址遍历handlerMappings获取对应的mapping
         .concatMap(mapping -> mapping.getHandler(exchange))
         .next()
       //没有找到的话就会返回404
         .switchIfEmpty(createNotFoundError())
       //调用具体的业务方法
         .flatMap(handler -> invokeHandler(exchange, handler))
       //处理结果并返回
         .flatMap(result -> handleResult(exchange, result));
}
```

具体过程如下：

1、通过webflux的HandlerMapping方法获取HandlerAdapter放到ServerWebExchange的属性中

2、通过获取到HandlerAdapter后触发handle方法，得到HandlerResult

3、通过HandlerResult，触发handleResult，针对不同的返回类找到不同的HandlerResultHandler

## 三大组件

### HandlerMapping

用于将请求映射到对应的处理器上

handlermapping的实现类如下：

![HandlerMapping](.\pic\HandlerMapping.png)

RequestMappingHandlerMapping：采用注解的方式，用@RequestMapping注解的方法的时候会使用RequestMappingHandlerMapping处理器

RouterFunctionMapping：用于功能端点的路由，如采用如下方式则会使用RouterFunctionMapping处理器。

```java
@Bean
public RouterFunction<ServerResponse> hello(HelloWorldHandler handler) {

    return RouterFunctions.route(RequestPredicates.path("/hello"),handler::helloWorld);
}
```

SimpleUrlHandlerMapping:用于URI路径模式和Webhandler实例显示注册的SimpleUrlHandlerMapping处理器

### HandlerAdapter

真正负责请求处理的方法，用于调用映射到请求的处理程序，即找到处理程序的方法进行业务处理。

HandlerAdapter的实现类如下：

![HandlerAdapter](.\pic\HandlerAdapter.png)

SimpleHandlerAdapter:和SimpleUrlHandlerMapping相对应

RequestMappingHandlerAdapter:和RequestMappingHandlerMapping相对应

HandlerFunctionAdapter:和RouterFunctionMapping相对应

### HandlerResultHandler

处理来自处理程序调用的结果，并最终确定响应。

![HandlerResultHandler](.\pic\HandlerResultHandler.png)





