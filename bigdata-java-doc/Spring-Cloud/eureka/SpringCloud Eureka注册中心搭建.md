## SpringCloud Eureka注册中心搭建

Eureka 是 Netflix 公司开源的一个服务注册与发现的组件，和其他 Netflix 公司的服务组件（例如负载均衡、熔断器、网关等）一起，被 Spring Cloud 整合为 Spring Cloud Netflix 模块。eureka从2.0版本开始闭源，1.0版本目前仍然在维护中

在进行注册中心与服务搭建的时候，首先需要配置host文件，host文件目录如下：C:\Windows\System32\drivers\etc\hosts,在hosts配置文件中添加如下信息

#eureka 服务配置
127.0.0.1 eureka-server.dev.com

127.0.0.1 eureka.server8761.com
127.0.0.1 eureka.server8762.com
127.0.0.1 eureka.server8763.com



### Eureka注册中心单实例搭建

eureka注册中心的搭建还是比较简单的，这里只是贴出了主要代码和配置，更详细的代码参考github

#### Eureka Server的注册中心搭建

单实例的注册中心搭建只需要在类上面添加一个注解@EnableEurekaServer即可,关于这个注解的详细解析，后面会有专门的文章进行解释

1、注册中心的配置文件如下

```yaml
server:
  port: 8761 #指定运行端口
spring:
  application:
    name: eureka-server #指定服务名称
eureka:
  instance:
    hostname: eureka-server.dev.com #指定主机地址
  client:
    fetch-registry: false #指定是否要从注册中心获取服务（注册中心不需要开启）
    register-with-eureka: false #指定是否要注册到注册中心（注册中心不需要开启）
    service-url:
      #设置与Eureka Server交互的地址查询服务和注册服务都需要依赖这个地址（单机）。
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/


logging:
  level:
    root: debug
```

2、java代码

```java
@EnableEurekaServer
@SpringBootApplication
public class RookieEurekaServerApplication {


    public static void main(String[] args) {

        SpringApplication.run(RookieEurekaServerApplication.class, args);

    }
}

```

直接运行代码，然后在浏览器中访问http://eureka-server.dev.com:8761/,如果出现如下页面，即表示注册中心已经搭建完成了

![4832c500c5036d814ff2298ea2d6899](.\pic\4832c500c5036d814ff2298ea2d6899.png)

#### Eureka Client 服务

创建两个服务，一个作为生产者服务 rookie-springcloud-eureka-producer，一个作为消费者服务rookie-springcloud-eureka-consumer，其中消费者服务对外提供访问的接口，将用户的访问通过内部调用生产者服务并最终返回给用户结果。

##### 创建消费者服务

1、配置文件信息

```yaml
server:
  port: 10001 #指定运行端口
spring:
  application:
    name: eureka-consumer #指定服务名称
eureka:
  client:
    service-url:
      defaultZone: ${EUREKA_DEFAULT_ZONE:http://eureka-server.dev.com:8761/eureka}


logging:
  level:
    root: info
```

java代码示例

```java
@RestController
public class ConsumerController {

    private final Logger logger = LoggerFactory.getLogger(getClass());

    @Autowired
    private RestTemplate restTemplate;

    @GetMapping("/consumer/hello")
    public ResponseEntity<String> getId() {
        ResponseEntity<String> result = restTemplate.getForEntity("http://eureka-producer//producer/hello", String.class);
        String hello = result.getBody();
        logger.info("request: {}", hello);
        return ResponseEntity.ok(hello);
    }


}
```

```java
//EnableDiscoveryClient注解可以省略
@EnableDiscoveryClient
@SpringBootApplication
public class RookieEurekaConsumerServerApplication {


    @Bean
    @LoadBalanced
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }


    public static void main(String[] args) {
        SpringApplication.run(RookieEurekaConsumerServerApplication.class, args);
    }

}

```

LoadBalanced的注解的作用主要用来让RestTemplate有负载均衡的能力

##### 创建生产者服务

1、配置文件信息

```yaml
server:
  port: 9091 #指定运行端口
spring:
  application:
    name: eureka-producer #指定服务名称
eureka:
  client:
    service-url:
      defaultZone: ${EUREKA_DEFAULT_ZONE:http://eureka-server.dev.com:8761/eureka}


logging:
  level:
    root: debug
```

2、java 示例代码

```java
@RestController
public class ProducerController {


    private final Logger logger = LoggerFactory.getLogger(getClass());


    @GetMapping("/producer/hello")
    public ResponseEntity<String> hello() {
        logger.info("request: {}", "hello");
        return ResponseEntity.ok("hello eureka");
    }

}
```

```java
@EnableDiscoveryClient
@SpringBootApplication
public class RookieEurekaProducerServerApplication {


    public static void main(String[] args) {
        SpringApplication.run(RookieEurekaProducerServerApplication.class, args);
    }

}

```

#### 启动服务

将上面的三个应用分别进行打包，并启动三个命令行窗口，按照如下顺序进行启动

```
java -jar -Dspring.profiles.active=dev rookie-springcloud-eureka-server-1.0-SNAPSHOT.jar

java -jar -Dspring.profiles.active=dev rookie-springcloud-eureka-consumer-1.0-SNAPSHOT.jar

java -jar -Dspring.profiles.active=dev rookie-springcloud-eureka-producer-1.0-SNAPSHOT.jar
```

在浏览器中访问  http://eureka-server.dev.com:8761/  即可看到 生产者和消费者已经注册到注册中心了，同时使用postman或者在浏览器中访问http://localhost:10001/consumer/hello,返回hello eureka即代表已经成功。

### Eureka注册中心多实例搭建

多实例集群的搭建和前面单实例搭建一样，只不过需要修改配置文件即可

#### 注册中心

将应用进行打包，然后启动windows的三个命令行窗口分别运行如下命令：

```
java -jar -Dspring.profiles.active=8761 rookie-springcloud-eureka-server-1.0-SNAPSHOT.jar

java -jar -Dspring.profiles.active=8762 rookie-springcloud-eureka-server-1.0-SNAPSHOT.jar

java -jar -Dspring.profiles.active=8763 rookie-springcloud-eureka-server-1.0-SNAPSHOT.jar
```



#### Eureka Client服务

```
java -jar -Dspring.profiles.active=9091 rookie-springcloud-eureka-producer-1.0-SNAPSHOT.jar

java -jar -Dspring.profiles.active=9092 rookie-springcloud-eureka-producer-1.0-SNAPSHOT.jar

java -jar -Dspring.profiles.active=9093 rookie-springcloud-eureka-producer-1.0-SNAPSHOT.jar

java -jar -Dspring.profiles.active=1001 rookie-springcloud-eureka-consumer-1.0-SNAPSHOT.jar
```

启动完成后，分别访问 http://eureka-server.dev.com:8761/，http://eureka-server.dev.com:8762/,http://eureka-server.dev.com:8763/，如图：

![4a124b193db1d88544fe95b22f17661](.\pic\4a124b193db1d88544fe95b22f17661.png)

访问http://localhost:10001/consumer/hello，会交替返回 hello euureka port:9091;hello euureka port:9092;hello euureka port:9093;这里就完成了集群的搭建即服务调用的负载均衡，因为有@LoadBalanced注解的作用，所以会交替输出该信息。