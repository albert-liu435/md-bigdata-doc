ThreadLocal用法

ThreadLocal的实例代表了一个线程局部的变量，每条线程都只能看到自己的值，并不会意识到其它的线程中也存在该变量。ThreadLocal是为了解决多线程访问同一个共享变量的时候容器出现并发问题。 ThreadLocal是除了加锁这种同步方式之外的一种保证一种规避多线程访问出现线程不安全的方法，当我们在创建一个变量后，如果每个线程对其进行访问的时候访问的都是线程自己的变量这样就不会存在线程不安全问题。 

ThreadLocal使用

```java
package com.rookie.bigdata.threadlocal;

/**

 * @author rookie

 * @version 1.0

 * @date 2020/3/24 23:50
   */
   public class ThrealLocalMain {

   private static ThreadLocal<String> threadLocal=new ThreadLocal<>();

   public static void main(String[] args) {
       Thread thread1=new Thread(new Runnable() {
           @Override
           public void run() {
               //设置线程本地变量的值
               threadLocal.set("zhangsan");

               hello("张三媳妇");
               System.out.println(":"+threadLocal.get());
       
           }
       });
       Thread thread2=new Thread(new Runnable() {
           @Override
           public void run() {
               threadLocal.set("lisi");
       
               hello("李四媳妇");
               System.out.println(":"+threadLocal.get());
       
           }
       });
       
       thread1.start();
       thread2.start();

   }

   public static void hello(String name){
       //打印当前线程本地内存中本地变量的值
       System.out.println(name+": "+threadLocal.get());
       //可以把这个注释掉
       threadLocal.remove();
   }

}
```

ThreadLocal源码分析

```java
public void set(T value) {
    //获取当前线程
    Thread t = Thread.currentThread();
    //以当前线程作为key值，去查找对应的线程变量，找到对应的map
    ThreadLocalMap map = getMap(t);
    //如果map不为null，就直接添加本地变量，key为当前线程，值为添加的本地变量值
    if (map != null)
        map.set(this, value);
    else
        //如果map为null，说明首次添加，需要首先创建出对应的map
        createMap(t, value);
}
```
```java
public T get() {
    //获取当前线程
    Thread t = Thread.currentThread();
    //获取当前线程的threadLocals变量
    ThreadLocalMap map = getMap(t);
    //如果threadLocals变量不为null，就可以在map中查找到本地变量的值
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    //threadLocals为null，调用该更改初始化当前线程的threadLocals变量
    return setInitialValue();
}
```
```java
 private T setInitialValue() {
        T value = initialValue();
     //获取当前线程
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
        return value;
    }
```

remove方法

 　remove方法判断该当前线程对应的threadLocals变量是否为null，不为null就直接删除当前线程中指定的threadLocals变量 

```java
 public void remove() {
     ThreadLocalMap m = getMap(Thread.currentThread());
     if (m != null)
         //如果map不为null，就移除当前线程中指定ThreadLocal实例的本地变量
         m.remove(this);
 }
```