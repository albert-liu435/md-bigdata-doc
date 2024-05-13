## java SPI机制

1、 SPI是Service Provider Interfaces的简称。根据Java的SPI规范，我们可以定义一个服务接口，具体的实现由对应的实现者去提供，即Service Provider（服务提供者）。然后在使用的时候只要根据SPI的规范去获取对应的服务提供者的服务实现即可.通过读取写入实现类的全限定名的配置文件来加载对应类，可以在运行的时候动态为接口替换实现类

SPI机制的约定：

1) 在META-INF/services/目录中创建以接口全限定名命名的文件该文件内容为Api具体实现类的全限定名

2) 使用ServiceLoader类动态加载META-INF中的实现类

3) 如SPI的实现类为Jar则需要放在主程序classPath中

4) Api具体实现类必须有一个不带参数的构造方法 

具体实现代码

### 定义SPIService接口

```java
public interface SPIService {

    void print(String msg);
}
```

两个实现类

```java
public class BJService implements SPIService {
    @Override
    public void print(String msg) {
        System.out.println("北京市");
    }
}

public class TJService implements SPIService {
    @Override
    public void print(String msg) {
        System.out.println("天津市");
    }
}
```

配置com.rookie.bigdata.util.spi.SPIService文件

```properties
#BJ
com.rookie.bigdata.util.spi.BJService
#TJ
com.rookie.bigdata.util.spi.TJService
```

直接进行测试即可.会打印出北京市和天津市

```java
@Test
public void print() {
    ServiceLoader<SPIService> serviceLoader = ServiceLoader.load(SPIService.class);
    SPIService spiService = null;
    for (SPIService service : serviceLoader) {

        service.print("SPI");
    }
}
```

相关源码[github](https://github.com/albert-liu435/rookies-javases/tree/master/rookie-javase/src/main/java/com/rookie/bigdata/util/spi)