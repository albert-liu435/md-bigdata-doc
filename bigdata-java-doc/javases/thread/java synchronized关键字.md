java synchronized关键字

### synchronized的使用

synchronized可以用来同步静态方法，普通方法和代码块。

修饰静态方法：当前类的Class对象当锁

修饰普通方法：类的当前对象this对象当锁

修饰代码块：自定义锁，放synchronized括号内

synchronized属于互斥锁，也是可重入锁在多线程操作被synchronized修饰的方法或者代码块的时候，首先需要获取锁，然后才能去执行里面的逻辑。这就保证了里面的逻辑在同一时刻只能被其中的一个线程去执行，不能同时执行。同时线程在获取锁前，会将自己的工作内存清除，从而保证变量副本从主内存中获取最新的值，而在线程释放锁的时候，会将工作内存的变量的值刷新到主内存。通过这些来保证线程操作的安全性。

```java
public class SynchronizedDemoTest {


    public Object object=new Object();

    //修饰静态方法
    public  synchronized static void print1(){
        System.out.println("hello synchronized print1" );
    }



    public void print2(){
        //自定义代码块
        synchronized (object){
            System.out.println("hello synchronized print2" );
        }
    }


    //修饰普通方法
    public synchronized void print3(){
        System.out.println("hello synchronized print3");
    }


    public static void main(String[] args) {


        SynchronizedDemoTest.print1();

        SynchronizedDemoTest synchronizedDemoTest=new SynchronizedDemoTest();
        synchronizedDemoTest.print2();
        synchronizedDemoTest.print3();


    }


}
```



### synchronized的原理

同步代码块：对于同步代码块，通过monitorenter与monitorexit指令实现，当线程执行到monitorenter的时候需要获取锁，当线程执行到monitorexit或异常钱需要释放锁。

同步方法：对于同步方法，通过ACC_SYNCHRONIZED标识,是一种隐士声明，底层也是monitor锁

### java对象头

Java对象头和monitor是实现synchronized的基础。synchronized用的锁是存在Java对象头里。



