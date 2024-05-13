#### JVM类加载机制

JVM类加载机制分为：加载，验证，准备，解析，初始化五步，如 下图：

![jvm加载](..\JVM\jvm加载.jpg)

- 加载：这个阶段会在内存中生成一个代表这个类的java.lang.Class对象作为方法区这个类的各种数据的入口。
- 验证：目的去报Class文件的字节流中包含的信息符合当前虚拟机的要求，并且不会危害虚拟机的自身安全。
- 准备：该阶段为类变量分配内存并设置类变量的初始值，即在 方法区中分配这些变量所使用的内存空间。
- 解析：指虚拟机将常量池中的符号引用替换为直接引用的过程(比如说方法的符号引用，是有方法名和相关描述符组成，在解析阶段，JVM把符号引用替换成一个指针，这个指针就是直接引用，它指向该类的该方法在方法区中的内存位置)
- 初始化：为类的静态变量赋予正确的初始值。当静态变量的等号右边的值是一个常量表达式时，不会调用static代码块进行初始化。只有等号右边的值是一个运行时运算出来的值，才会调用static初始化.

以下几种情况不会执行初始化操作

- 通过子类引用父类的静态字段，只会触发父类的初始化，而不会触发子类的初始化
- 定义对象数组，不会触发该类的初始化
- 常量在编译期间会存入调用类的常量池中，本质上并没有直接引用定义常量的类，不会触
  发定义常量所在的类
- 通过类名获取Class 对象，不会触发类的初始化
- 通过Class.forName 加载指定类时，如果指定参数initialize 为false 时，也不会触发类初
  始化，其实这个参数是告诉虚拟机，是否要对类进行初始化
- 通过ClassLoader 默认的loadClass 方法，也不会触发初始化动作

#### 类加载器

JVM提供了三种类型的加载器

- 启动类加载器：负责加载JAVA_HOME/lib目录中的，或者通过-Xbootclasspath参数指定路径中的，且被虚拟机认可的类
- 扩展类加载器：负责加载 JAVA_HOME\lib\ext 目录中的，或通过java.ext.dirs 系统变量指定路径中的类
  库
- 应用程序类加载器：负责加载用户路径（classpath）上的类库。

JVM 通过双亲委派模型进行类的加载，当然我们也可以通过继承java.lang.ClassLoader
实现自定义的类加载器

![JVM类加载机制](..\JVM\JVM类加载机制.bmp)

```java
public class ClassLoaderMain {

    public static void main(String[] args) {
        ClassLoader contextClassLoader = Thread.currentThread().getContextClassLoader();

//当前的ClassLoader是AppClassLoder，其父ClassLoader是ExtClassLoader，祖父ClassLoader是根类装载器。java无法获取它的句柄，所以返回是null
        System.out.println("1  "+contextClassLoader);
        System.out.println("2  "+contextClassLoader.getParent());
        System.out.println("3  "+contextClassLoader.getParent().getParent());
    }

}
```

运行上面程序即可发现， 当前的ClassLoader是AppClassLoder，其父ClassLoader是ExtClassLoader，祖父ClassLoader是根类装载器。java无法获取它的句柄，所以返回是null 。

###### 双亲委派

当一个类收到了类加载请求，他首先不会尝试自己去加载这个类，而是把这个请求委派给父类去完成，每一个层次类加载器都是如此，因此所有的加载请求都应该传送到启动类加载其中，只有当父类加载器反馈自己无法完成这个请求的时候（在它的加载路径下没有找到所需加载的
Class），子类加载器才会尝试自己去加载.

采用双亲委派的一个好处是比如加载位于rt.jar 包中的类java.lang.Object，不管是哪个加载器加载这个类，最终都是委托给顶层的启动类加载器进行加载，这样就保证了使用不同的类加载器最终得到的都是同样一个Object 对象

#### ClassLoader的重要方法

- Class LoadClass(String name):name参数指定类装载器需要装载类的名字，为全限定类名。如com.rookie.bigdata.reflect.Person。并且存在一个重载方法loadClass(String name,boolean resolve);resolve用来用来标识是否需要解析该类。在初始化之前，应考虑进行类解析工作，但并不是所有的类都需要解析。如果JVM只需要知道该类是否存在或者找出该类的超类，那么久不需要进行解析。
- Class DefineClass(String name,byte[] b,int off,int len):将类文件的字节数组转换成JVM内部的java.lang,Class对象。字节数组可以从本地文件系统，远程网络中获取。参数name为字节数组对应的全限定类名。
- Class findSystemClass(String name):从本地文件系统载入Class文件。该方法是JVM默认使用的装载机制。
- Class findLoadedClass(String name):调用该方法来查看ClassLoader是否已装入某个类，如果已装入，那么返回java.lang.Class对象；否则返回null.
- ClassLoader getParent():获取类装载器的父装载器，除根装载器外，所有的类装载器都有且仅有一个父装载器。ExtClassLoader的父装载器是根装载器，因为根装载器是非java语言编写，所以无法获得。

参考：https://segmentfault.com/a/1190000014395186