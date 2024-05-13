## log4j

log4j是apache的一个开源项目，是一个功能强大的日志组件，提供方便的日志记录。虽然我们现在已经合并到log4j2,但是并不妨碍我们去学习并了解它。

### log4j的日志等级

log4j的日志等级包装在Level类中，主要包括如下几个等级

```java
  final static public Level OFF = new Level(OFF_INT, "OFF", 0);

  final static public Level FATAL = new Level(FATAL_INT, "FATAL", 0);

  final static public Level ERROR = new Level(ERROR_INT, "ERROR", 3);

  final static public Level WARN  = new Level(WARN_INT, "WARN",  4);

  final static public Level INFO  = new Level(INFO_INT, "INFO",  6);

  final static public Level DEBUG = new Level(DEBUG_INT, "DEBUG", 7);

  public static final Level TRACE = new Level(TRACE_INT, "TRACE", 7);


  final static public Level ALL = new Level(ALL_INT, "ALL", 7);
```

日志等级从高到底如下：
FATAL <- ERROR <- WARN <- INFO <- DEBUG <-TRACE
FATAL:严重错误，一般会造成系统的崩溃和终止运行
ERROR:错误信息
WARN：警告信息
INFO:程序运行的信息
DEBUG：调试信息，一般在程序调试阶段会开启该模式
TARCE:追踪信息，记录程序的所有的流程信息

其中OFF和ALL分别代表关闭日志记录和启用所有的日志记录

### log4j的日志输出

log4j主要靠 Logger,Appender, Layout三大组件将日志信息输出到控制台，文件以及网络中
Logger:用于收集处理日志记录，其中包含一个RootLogger，其他logger会直接或间接继承该RootLogger.

![1640849417](.\pic\1640849417.png)

Appender：用于指定日志输出的目的地，Appender的实现类如下：

JDBCAppender：把日志保存到数据库中

FileAppender:把日志输出到文件中

ConsoleAppender:把日志输出到控制台中

DailyRollingFileAppender:将日志输出到一个日志文件，并且每天输出到一个新的文件,也可以自定义按照每小时或者每分钟输出一个日志文件

RollingFileAppender：将日志信息输出到一个日志文件，并且指定文件的尺寸，当文件大小达到指定尺寸时，会自动把文件改名，同时产生一个新的文件

SMTPAppender：日志信息可以通过邮件发送出去

SocketAppender：将日志信息发送到网络中

![62e2a91e922d47941ec5a0653c3f7c0](.\pic\62e2a91e922d47941ec5a0653c3f7c0.png)

Layout:用于控制日志的输出格式，Layout的实现类如下：

HTMLLayout:格式化日志输出为HTML表格形式

SimpleLayout:简单的日志输出格式化，

PatternLayout:最强大的格式化期，可以根据自定义格式输出日志，如果没有指定转换格式， 就是用默认的转换格式

### log4j配置文件详解

```properties
#log4j的配置详细配置

#第一个参数为日志等级,分别为OFF、FATAL、ERROR、WARN、INFO、DEBUG、ALL，第二个参数为指定的appender的名称，即跟后面结合确定日志输出的位置,格式如下
#log4j.rootLogger=level,appenderName,appenderName1,....
#例如:
#log4j.rootLogger=debug,console

#配置appender,即日志输出的目的地,如控制台，文件中等等
#log4j.appender.appenderName=log4j提供的appender类，也可以根据自己的需要实现自己的appender类
#例如
#log4j.appender.console=org.apache.log4j.ConsoleAppender


#appender主要有如下几种
#org.apache.log4j.ConsoleAppender（控制台）
#org.apache.log4j.FileAppender（文件）
#org.apache.log4j.DailyRollingFileAppender（每天产生一个日志文件）
#org.apache.log4j.RollingFileAppender（文件大小到达指定尺寸的时候产生一个新的文件）
#org.apache.log4j.WriterAppender（将日志信息以流格式发送到任意指定的地方）

#ConsoleAppender
#Threshold=INFO：指定日志消息的输出层次。
#ImmediateFlush=true：默认值是true,意谓着所有的消息都会被立即输出。
#Target=System.err：默认情况下是：System.out,指定输出控制台
#例如:
#log4j.appender.console.Threshold=INFO
#log4j.appender.console.ImmediateFlush=true
#log4j.appender.console.Target=System.err

#FileAppender
#Threshold=INFO：指定日志消息的输出层次。
#ImmediateFlush=true：默认值是true,意谓着所有的消息都会被立即输出。
#File=mylog.log：指定消息输出到mylog.log文件。
#Append=false：默认值是true,即将消息增加到指定文件中，false指将消息覆盖指定的文件内容。


#DailyRollingFileAppender
#Threshold=INFO：指定日志消息的输出层次。
#ImmediateFlush=true：默认值是true,意谓着所有的消息都会被立即输出。
#File=mylog.log：指定消息输出到mylog.log文件。
#Append=false：默认值是true,即将消息增加到指定文件中，false指将消息覆盖指定的文件内容。
#DatePattern=”.”yyyy-ww:每周滚动一次文件，即每周产生一个新的文件。
#1)”.”yyyy-MM: 每月
#2)”.”yyyy-ww: 每周
#3)”.”yyyy-MM-dd: 每天
#4)”.”yyyy-MM-dd-a: 每天两次
#5)”.”yyyy-MM-dd-HH: 每小时
#6)”.”yyyy-MM-dd-HH-mm: 每分钟


#RollingFileAppender
#Threshold=INFO：指定日志消息的输出层次。
#ImmediateFlush=true：默认值是true,意谓着所有的消息都会被立即输出。
#File=mylog.log：指定消息输出到mylog.log文件。
#Append=false：默认值是true,即将消息增加到指定文件中，false指将消息覆盖指定的文件内容。
#MaxFileSize=100KB：后缀可以是KB, MB 或者是 GB. 在日志文件到达该大小时，将会自动滚动，即将原来的内容移到mylog.log.1文件。
#MaxBackupIndex=5：指定可以产生的滚动文件的最大数。



##Layout的配置格式如下
#log4j.appender.appenderName.layout=value  Log4j提供的layout类,也可以是自己的实现类
#log4j.appender.appenderName.layout.属性 = 值

##Layout主要有如下几种
#org.apache.log4j.HTMLLayout（以HTML表格形式布局），
#org.apache.log4j.PatternLayout（可以灵活地指定布局模式），
#org.apache.log4j.SimpleLayout（包含日志信息的级别和信息字符串），
#org.apache.log4j.TTCCLayout（包含日志产生的时间、线程、类别等等信息）


#HTMLLayout
#LocationInfo=true：默认值是false,输出java文件名称和行号
#Title=my app file：默认值是 Log4J Log Messages
#
#PatternLayout
#ConversionPattern=%m%n：指定怎样格式化指定的消息。
#
#XMLLayout
#LocationInfo=true：默认值是false,输出java文件和行号
#Log4J采用类似C语言中的printf函数的打印格式格式化日志信息，打印参数如下：

## 占位符相关含义
#%p: 输出优先级，及 DEBUG、INFO 等
#%m: 输出代码中指定的日志信息
#%n: 换行符（Windows平台的换行符为 "\n"，Unix 平台为 "\n"）
#%r: 输出自应用启动到输出该 log 信息耗费的毫秒数
#%d: 输出服务器当前时间，默认为 ISO8601，也可以指定格式，如：%d{yyyy-MM-dd HH:mm:ss:SSS}  => 年月日 时分秒毫秒
#
#%l: 输出日志时间发生的位置，包括类名、线程、及在代码中的行数。如：Test.main(Test.java:10)
## %l即可表示下方四个修饰符
#%c: 输出打印语句所属的类的全名
#%t: 输出产生该日志的线程全名
#%F: 输出日志消息产生时所在的文件名称
#%L: 输出代码中的行号
#%%: 输出一个 "%" 字符
#
## 可在例如%m之间加入修饰符来控制最小宽度、最大宽度和文本的对其方式
#%5c: 输出category名称，最小宽度是5，category<5，默认的情况下右对齐
#%-5c: 输出category名称，最小宽度是5，category<5，"-"号指定左对齐,会有空格
#%.5c: 输出category名称，最大宽度是5，category>5，就会将左边多出的字符截掉，<5不会有空格
#%20.30c: category名称<20补空格，并且右对齐，>30字符，就从左边交远销出的字符截掉

```

### log4j示例及执行流程

示例代码及配置如下：

```java
    @Test
    public void test2() {
        //log4j的初始化配置，如果resources中没有log4j.properties文件的情况下，如果不配置则会报错
        // BasicConfigurator.configure();

        Logger logger = Logger.getLogger(log4jMain.class);
        logger.fatal("fatal");
        logger.error("error");
        logger.warn("warn");
        logger.info("info");
        logger.debug("debug");
        logger.trace("trace");


    }
	
	
#第一个参数为日志等级，第二个参数为指定的appender
log4j.rootLogger=debug,console


log4j.com.rookie.bigdata.level=info
log4j.appender.console=org.apache.log4j.ConsoleAppender
#输出日志的格式即布局
log4j.appender.console.layout = org.apache.log4j.PatternLayout
#自定义日志格式，即控制台输出的日志格式
log4j.appender.console.layout.ConversionPattern = [%-5p] %d{yyyy-MM-dd HH:mm:ss,SSS} method:%l%n%m%n

#按照级别输出日志，大于该级别的日志会进行输出
log4j.appender.console.threshold=info
```

源码分析

```java
//org.apache.log4j.LogManager 
static {
    
	//省略代码....
	//这里会按照顺序来加载log4j的配置文件，首先会加载log4j.xml配置文件，如果不存在的话才会加载log4j.properties配置文件
	//也就是说如果两者都存在的情况下，会采用log4j.xml文件的配置方式
      if(configurationOptionStr == null) {	
	url = Loader.getResource(DEFAULT_XML_CONFIGURATION_FILE);
	if(url == null) {
	  url = Loader.getResource(DEFAULT_CONFIGURATION_FILE);
	}
      } else {
	try {
	  url = new URL(configurationOptionStr);
	} catch (MalformedURLException ex) {
	  // so, resource is not a URL:
	  // attempt to get the resource from the class path
	  url = Loader.getResource(configurationOptionStr); 
	}	
      }

      if(url != null) {
	    LogLog.debug("Using URL ["+url+"] for automatic log4j configuration.");
        try {
			//这里会对配置文件进行解析
            OptionConverter.selectAndConfigure(url, configuratorClassName,
					   LogManager.getLoggerRepository());
        } catch (NoClassDefFoundError e) {
            LogLog.warn("Error during default initialization", e);
        }
      } else {
	    LogLog.debug("Could not find resource: ["+configurationOptionStr+"].");
      }
    } else {
        LogLog.debug("Default initialization of overridden by " + 
            DEFAULT_INIT_OVERRIDE_KEY + "property."); 
    }  
  } 
  
//org.apache.log4j.helpers.OptionConverter
    static
  public
  void selectAndConfigure(URL url, String clazz, LoggerRepository hierarchy) {
   Configurator configurator = null;
   String filename = url.getFile();
	//如果为xml文件,则采用DOMConfigurator进行解析,并实例化log4j相关的组件
   if(clazz == null && filename != null && filename.endsWith(".xml")) {
     clazz = "org.apache.log4j.xml.DOMConfigurator";
   }

   if(clazz != null) {
     LogLog.debug("Preferred configurator class: " + clazz);
     configurator = (Configurator) instantiateByClassName(clazz,
							  Configurator.class,
							  null);
     if(configurator == null) {
   	  LogLog.error("Could not instantiate configurator ["+clazz+"].");
   	  return;
     }
   } else {
	   //如果是property文件，则采用PropertyConfigurator进行解析,并实例化log4j相关的组件
     configurator = new PropertyConfigurator();
   }

   configurator.doConfigure(url, hierarchy);
  }
}
```

上面初始化完成之后，会进行日志输出，下面以log.info(”info“)为例说明

```java
  //org.apache.log4j.Category
  public
  void info(Object message) {
	  //会进行判断，如果为true,则不将信息输出，默认情况下都是false,
    if(repository.isDisabled(Level.INFO_INT))
      return;
	//判断当前的级别是否大于设置的级别，如果为true,则进行日志输出
    if(Level.INFO.isGreaterOrEqual(this.getEffectiveLevel()))
		//日志输出
      forcedLog(FQCN, Level.INFO, message, null);
  }
  
    protected
  void forcedLog(String fqcn, Priority level, Object message, Throwable t) {
	  //封装成一个log的event事件并进行输出
    callAppenders(new LoggingEvent(fqcn, this, level, message, t));
  }
  
  
    public
  void callAppenders(LoggingEvent event) {
    int writes = 0;

    for(Category c = this; c != null; c=c.parent) {
      // Protected against simultaneous call to addAppender, removeAppender,...
      synchronized(c) {
	if(c.aai != null) {
		//最终会调用org.apache.log4j.helpers.AppenderAttachableImpl类中的方法
	  writes += c.aai.appendLoopOnAppenders(event);
	}
	if(!c.additive) {
	  break;
	}
      }
    }

    if(writes == 0) {
      repository.emitNoAppenderWarning(this);
    }
  }
  
  //org.apache.log4j.helpers.AppenderAttachableImpl
  
    public
  int appendLoopOnAppenders(LoggingEvent event) {
    int size = 0;
    Appender appender;

    if(appenderList != null) {
      size = appenderList.size();
      for(int i = 0; i < size; i++) {
	appender = (Appender) appenderList.elementAt(i);
	//调用 org.apache.log4j.AppenderSkeleton中的方法
	appender.doAppend(event);
      }
    }    
    return size;
  }
  
  //org.apache.log4j.WriterAppender
    protected
  void subAppend(LoggingEvent event) {
	  //这里会将打印的信息输出到控制台
    this.qw.write(this.layout.format(event));

    if(layout.ignoresThrowable()) {
      String[] s = event.getThrowableStrRep();
      if (s != null) {
	int len = s.length;
	for(int i = 0; i < len; i++) {
	  this.qw.write(s[i]);
	  this.qw.write(Layout.LINE_SEP);
	}
      }
    }

    if(shouldFlush(event)) {
      this.qw.flush();
    }
  }
```

