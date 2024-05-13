java自动自动拆装箱

最近在看Effective-Java-3rd发现其中介绍一个自动拆装箱影响程序的性能问题。

自动拆装箱是java 1.5开始新增的功能，它允许程序员混合基本类型和包装类型，根据需要自动装箱和拆箱。但是在性能上面还是有较大的差别的。看下面的例子：

```java
public static void main(String[] args) {

    long startTime=System.currentTimeMillis();
    System.out.println(sum());
    System.out.println("共耗时："+(System.currentTimeMillis()-startTime)+"毫秒");
}

private static long sum() {
    //这个比较性能较差
    //Long sum = 0L;
    //这个性能较好
    long sum = 0L;
    for (long i = 0; i <= Integer.MAX_VALUE; i++)
        sum += i;
    return sum;
}
```
运行结果

```java
累加值为：2305843008139952128
共耗时：872毫秒
```

而采用包装类进行相加时，例子如下

```java
public class LongMain {

    public static void main(String[] args) {
    
        long startTime=System.currentTimeMillis();
        System.out.println("累加值为："+sum());
        System.out.println("共耗时："+(System.currentTimeMillis()-startTime)+"毫秒");
    }
    
    private static long sum() {
        //这个比较性能较差
        Long sum = 0L;
        //这个性能较好
        //long sum = 0L;
        for (long i = 0; i <= Integer.MAX_VALUE; i++)
            sum += i;
        return sum;
    }

}
```

运行结果如下：

```java
累加值为：2305843008139952128
共耗时：12337毫秒
```

可以看出包装类与非包装类之间的运行时间相差将近14倍，究其原因，是每次包装类型的Long在进行sum+=1时，都会创建一个新的对象，所以导致性能下降。

结论：

从性能的方面考虑，基本类型优于包装类，优先使用基本类型，应提防意外的自动装箱。