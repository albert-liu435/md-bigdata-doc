SLF4J的使用

SLF4J是一个日志门面，主要为java日志访问提供一套标准、规范的额API框架，主要提供的是接口，具体的实现交由其他的框架，如log4j和logback等。slf4j-api作为门面，皮之尚具体的实现框架(log4j,logback等)，中间使用桥接器完成桥接

SLF4J中最重要的两个类为org.slf4j.Logger和org.slf4j.LogFactory

SLF4J框架的好处

使用SLF4J框架，可以在部署时迁移到所需的日志记录框架

SLF4J提供了对所有流行的日志框架的绑定，例如log4j，JUL，Simple logging和NOP。因此可以在部署时切换到任何这些流行的框架。

无论使用哪种绑定，SLF4J都支持参数化日志记录消息。由于SLF4J将应用程序和日志记录框架分离，因此可以轻松编写独立于日志记录框架的应用程序。而无需担心用于编写应用程序的日志记录框架。

SLF4J提供了一个简单的Java工具，称为迁移器。使用此工具，可以迁移现有项目，这些项目使用日志
框架(如Jakarta Commons Logging(JCL)或log4j或Java.util.logging(JUL))到SLF4J。

SLF4J绑定日志的实现

下面的官网的SLF4J的绑定实现图，可以通过无缝的对其他框架进行切换

![concrete-bindings](.\pic\concrete-bindings.png)

初始绑定日志实现原理

首先调用如下代码

```java
public final static Logger LOGGER = LoggerFactory.getLogger(Slf4jSimpleMain.class);
```

LoggerFactory.java

```java
    public static Logger getLogger(Class<?> clazz) {
    //调用重载方法，传入类名来获取Logger实例
        Logger logger = getLogger(clazz.getName());
        if (DETECT_LOGGER_NAME_MISMATCH) {
            Class<?> autoComputedCallingClass = Util.getCallingClass();
            if (autoComputedCallingClass != null && nonMatchingClasses(clazz, autoComputedCallingClass)) {
                Util.report(String.format("Detected logger name mismatch. Given name: \"%s\"; computed name: \"%s\".", logger.getName(),
                                autoComputedCallingClass.getName()));
                Util.report("See " + LOGGER_NAME_MISMATCH_URL + " for an explanation");
            }
        }
        return logger;
    }

    //根据类名来获取logger实例
    public static Logger getLogger(String name) {
        //获取ILoggerFactory接口的实现类(接口方法是getLogger()),这里为SimpleLoggerFactory实例
        ILoggerFactory iLoggerFactory = getILoggerFactory();
        return iLoggerFactory.getLogger(name);
    }

    public static ILoggerFactory getILoggerFactory() {
        if (INITIALIZATION_STATE == UNINITIALIZED) {
            synchronized (LoggerFactory.class) {
                if (INITIALIZATION_STATE == UNINITIALIZED) {
                    INITIALIZATION_STATE = ONGOING_INITIALIZATION;
                    //进行初始化操作
                    performInitialization();
                }
            }
        }
        switch (INITIALIZATION_STATE) {
        case SUCCESSFUL_INITIALIZATION:
            return StaticLoggerBinder.getSingleton().getLoggerFactory();
        case NOP_FALLBACK_INITIALIZATION:
            return NOP_FALLBACK_FACTORY;
        case FAILED_INITIALIZATION:
            throw new IllegalStateException(UNSUCCESSFUL_INIT_MSG);
        case ONGOING_INITIALIZATION:
            // support re-entrant behavior.
            // See also http://jira.qos.ch/browse/SLF4J-97
            return SUBST_FACTORY;
        }
        throw new IllegalStateException("Unreachable code");
    }

    private final static void performInitialization() {
        //进行绑定操作
        bind();
        if (INITIALIZATION_STATE == SUCCESSFUL_INITIALIZATION) {
            versionSanityCheck();
        }
    }
	//进行绑定操作
    private final static void bind() {
        try {
            Set<URL> staticLoggerBinderPathSet = null;
            // skip check under android, see also
            // http://jira.qos.ch/browse/SLF4J-328
            if (!isAndroid()) {
                //查找可能的静态日志执行器的路径
                staticLoggerBinderPathSet = findPossibleStaticLoggerBinderPathSet();
                //超过一个的话进行控制台输出警告
                reportMultipleBindingAmbiguity(staticLoggerBinderPathSet);
            }
            // the next line does the binding
            StaticLoggerBinder.getSingleton();
            INITIALIZATION_STATE = SUCCESSFUL_INITIALIZATION;
            reportActualBinding(staticLoggerBinderPathSet);
        } catch (NoClassDefFoundError ncde) {
            String msg = ncde.getMessage();
            if (messageContainsOrgSlf4jImplStaticLoggerBinder(msg)) {
                INITIALIZATION_STATE = NOP_FALLBACK_INITIALIZATION;
                Util.report("Failed to load class \"org.slf4j.impl.StaticLoggerBinder\".");
                Util.report("Defaulting to no-operation (NOP) logger implementation");
                Util.report("See " + NO_STATICLOGGERBINDER_URL + " for further details.");
            } else {
                failedBinding(ncde);
                throw ncde;
            }
        } catch (java.lang.NoSuchMethodError nsme) {
            String msg = nsme.getMessage();
            if (msg != null && msg.contains("org.slf4j.impl.StaticLoggerBinder.getSingleton()")) {
                INITIALIZATION_STATE = FAILED_INITIALIZATION;
                Util.report("slf4j-api 1.6.x (or later) is incompatible with this binding.");
                Util.report("Your binding is version 1.5.5 or earlier.");
                Util.report("Upgrade your binding to version 1.6.x.");
            }
            throw nsme;
        } catch (Exception e) {
            failedBinding(e);
            throw new IllegalStateException("Unexpected initialization failure", e);
        } finally {
            postBindCleanUp();
        }
    }
```

