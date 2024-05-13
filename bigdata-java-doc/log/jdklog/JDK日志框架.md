JDK日志框架

### JDK Logging

JDK自带的日志框架是从JDK1.4开始，不需要任何类库的支持，JDK logging相关的类位于java.util.logging包下面。我们虽然在开发的时候不怎么使用这个JDK自带的日志框架，但是许多其他的框架在编写的时候都对他进行了支持，如tomcat中的DirectJDKLog，mybatis中的Jdk14LoggingImpl。

Logging日志等级

logging的日志等级依次如下，等级依次变高，如当设置等级为INFO，那么比INFO等级低的日志将不会输出

ALL→FINEST→FINER→FINE→CONFIG→INFO→WARNING→SEVERE→OFF

### Logging配置

logging的配置主要如下

 handlers：当配置多个Handler时需要用逗号分隔每个Handler

.level:是root logger的日志级别，logger是有继承关系的，如配置文件中的com.xyz.foo.level = SEVERE,即com.xyz.foo包下面的打印的日志等级为SERVRE级别，如果没有其他特殊的配置，则com.xyz.foo.abc包下面的日志等级也是SEVERE级别

默认情况下，jdklogging的配置文件位置在$JAVA_HOME/jre/lib/logging.properties，配置信息如下：

```properties
#可以配置多个，当配置多个的时候用分割
handlers= java.util.logging.ConsoleHandler
# To also add the FileHandler, use the following line instead.
#handlers= java.util.logging.FileHandler, java.util.logging.ConsoleHandler
#①%h表示当前用户目录路径，②%u对  # 应下面的count，范围为[0,count)
java.util.logging.FileHandler.pattern = %h/java%u.log
#输出日志文件限制大小（50000字节）
java.util.logging.FileHandler.limit = 50000
# 定义上面%h的范围
java.util.logging.FileHandler.count = 1
# 设置FileHandler中的转换器为     XMLFormatter (XML格式)
java.util.logging.FileHandler.formatter = java.util.logging.XMLFormatter

# Limit the message that are printed on the console to INFO and above.
# handler的日志等级为INFO
java.util.logging.ConsoleHandler.level = INFO
# 该handler转换器为         SimpleForamtter
java.util.logging.ConsoleHandler.formatter = java.util.logging.SimpleFormatter

# For example, set the com.xyz.foo logger to only log SEVERE
# messages:
com.xyz.foo.level = SEVERE

```

配置文件的加载是在LogManager中的readConfiguration()方法中加载的，可以对其进行debug来运行查看

### java.util.logging包下的主要类和接口

LogManager：整个JVM内部所有logger的管理，logger的生成、获取等操作都依赖于它，也包括配置文件的读取

logger：日志记录器，通常为应用程序的入口，将打印的日志发送给Handlers去处理

handler：用于控制日志的输出，通过Formatters来格式化信息，将日志输出到控制台或者文件系统中或者网络中。JDK中的Handler的实现如下，我们也可以实现自己的Handler

StreamHandler：将日志通过OutputStream输出。
FileHandler：对应的OutputStream对象是FileOutputStream，能够将日志输出到文件。
ConsoleHandler：对应的OutputStream对象是System.err，会将日志输出到控制台。
SocketHandler：对应的OutputStream对象是Socket.getOutputStream()，会将日志输出到网络套接字。
MemoryHandler：将日志输出到一个内存缓冲区。

![1640338936](.\pic\1640338936.png)

Formatter:用于对日志进行格式化处理,包括 时间格式，包的名称，方法名，详细信息等。Formatter的继承如下：

![1640339155](.\pic\1640339155.png)

Filter：过滤器，根据需要来定制哪些日志会被记录或放过。

Level:是一个枚举类里面封装了日志的等级等信息

LogRecord:即日志的信息对象，里面封装了日志的等级、消息信息、线程号、时间millis等信息

### JDK logging示例

下面示例以$JAVA_HOME/jre/lib/logging.properties中的默认配置文件为例进行

```java
public class LoggerMain {

   public static final Logger logger = Logger.getLogger("com.java.util.logging");
    public static void main(String[] args) {

        logger.info("开始执行...");
        logger.warning("大于info级别...");
        logger.fine("小于info级别...");
        logger.severe("结束执行...");
    }
}

#打印的日志如下
十二月 24, 2021 6:13:02 下午 com.java.util.logging.LoggerMain main
信息: 开始执行...
十二月 24, 2021 6:13:02 下午 com.java.util.logging.LoggerMain main
警告: 大于info级别...
十二月 24, 2021 6:13:02 下午 com.java.util.logging.LoggerMain main
严重: 结束执行...
```

执行过程如下

1、获取logger实例

```java
public static final Logger logger = Logger.getLogger("com.java.util.logging");
```

2、对日志进行输出

```java
 public void log(Level level, String msg) {
 	//对日志级别进行校验，默认日志等级为INFO,小于当前等级的直接返回，不再进行日志输出
     if (!isLoggable(level)) {
         return;
     }
	 //封装成LogRecord对象
     LogRecord lr = new LogRecord(level, msg);
     doLog(lr);
 }
	//进行日志等级的判断
    public boolean isLoggable(Level level) {
        if (level.intValue() < levelValue || levelValue == offValue) {
            return false;
        }
        return true;
    }

    private void doLog(LogRecord lr) {
        lr.setLoggerName(name);
        final LoggerBundle lb = getEffectiveLoggerBundle();
        //进行国际化
        final ResourceBundle  bundle = lb.userBundle;
        final String ebname = lb.resourceBundleName;
        if (ebname != null && bundle != null) {
            lr.setResourceBundleName(ebname);
            lr.setResourceBundle(bundle);
        }
        log(lr);
    }

    public void log(LogRecord record) {
		//校验日志级别
        if (!isLoggable(record.getLevel())) {
            return;
        }
		//过滤操作
        Filter theFilter = filter;
        if (theFilter != null && !theFilter.isLoggable(record)) {
            return;
        }

        // Post the LogRecord to all our Handlers, and then to
        // our parents' handlers, all the way up the tree.

        Logger logger = this;
        while (logger != null) {
            final Handler[] loggerHandlers = isSystemLogger
                ? logger.accessCheckedHandlers()
                : logger.getHandlers();

            for (Handler handler : loggerHandlers) {
			//真正的对日志进行输出,这里会调用ConsoleHandler将日志输出到控制台
                handler.publish(record);
            }

            final boolean useParentHdls = isSystemLogger
                ? logger.useParentHandlers
                : logger.getUseParentHandlers();

            if (!useParentHdls) {
                break;
            }

            logger = isSystemLogger ? logger.parent : logger.getParent();
        }
    }

```

至此log日志已经打印到控制台了

### 自定义日志输出到文件中

日志配置文件

```properties

handlers= java.util.logging.ConsoleHandler,java.util.logging.FileHandler


.level= INFO

############################################################
# Handler specific properties.
# Describes specific configuration info for Handlers.
############################################################

# default file output is in user's home directory.
java.util.logging.FileHandler.pattern = %h/logs/java%u.log
#每个文件输出的字节数
java.util.logging.FileHandler.limit = 50000
#保留的日志文件的个数
java.util.logging.FileHandler.count = 10
java.util.logging.FileHandler.formatter = java.util.logging.XMLFormatter

# Limit the message that are printed on the console to INFO and above.
java.util.logging.ConsoleHandler.level = INFO
java.util.logging.ConsoleHandler.formatter = java.util.logging.SimpleFormatter



# For example, set the com.xyz.foo logger to only log SEVERE
# messages:
com.xyz.foo.level = SEVERE
```

示例代码

```java
        // 使用LogManager的单例来读取该配置文件(与源码读取官方logging.properties方法类似)
//        FileInputStream inputStream = new FileInputStream("C:\\work\\IDEAWorkSpace\\rookie-project\\haizhilangzigithub\\rookies-javases\\rookie-javase\\src\\main\\resources\\com\\java\\util\\logging\\logging.properties");  //注意路径
//
//        Properties properties=new Properties();
//        properties.load(inputStream);


        //使用System来配置文件
        System.setProperty("java.util.logging.config.file", "C:\\work\\IDEAWorkSpace\\rookie-project\\haizhilangzigithub\\rookies-javases\\rookie-javase\\src\\main\\resources\\com\\java\\util\\logging\\logging.properties");


        Logger logger = Logger.getLogger("com.java.util.logging");

        for (int i = 0; i < 1000000; i++) {
            logger.info("test" + i);
        }

```

