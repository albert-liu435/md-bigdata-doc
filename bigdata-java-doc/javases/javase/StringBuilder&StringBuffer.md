StringBuilder&StringBuffer

Stringbuffer是一个线程安全的可变的字符序列，即StringBuffer是线程安全的，而StringBuilder是非线程安全的。继承关系图如下：

![AbstractStringBuilder](.\dic\AbstractStringBuilder.png)

CharSequence:是一个接口，CharSequence是char值的可读序列。 该接口提供统一的，只读访问许多不同类型的char序列。 char值代表基本多语言平面（BMP）或代理中的一个字符。 详见Unicode Character Representation 。此接口不会完善equals和hashCode方法。 因此，比较两个对象实现CharSequence其结果是，在一般情况下，不确定的。 每个对象可以由不同的类实现，并且不能保证每个类都能够测试其与另一个类相同。 因此，使用任意的CharSequence实例作为集合中的元素或映射中的键是不合适的。

Appendable:也是一个接口，能够被添加 char 序列和值的对象。如果某个类的实例打算接收取自 Formatter 的格式化输出，那么该类必须实现 Appendable 接口。要添加的字符应该是有效的 Unicode 字符，正如 Unicode Character Representation 中描述的那样。注意，增补字符可能由多个 16 位 char 值组成。Appendable 对于多线程访问而言没必要是安全的。线程安全由扩展和实现此接口的类负责。由于此接口可能由具有不同的错误处理风格的现有类实现，所以无法保证错误不会传播给调用者。

StringBuffer:线程安全的可变的字符序列， 字符串缓冲区就像一个String ，但可以修改。 在任何时间点，它包含一些特定的字符序列，但可以通过某些方法调用来更改序列的长度和内容 。 字符串缓冲区可以安全地被多个线程使用。 这些方法在必要时进行同步，以便任何特定实例上的所有操作都按照与所涉及的各个线程所执行的方法调用顺序一致的顺序发生 。

StringBuilder:一个可变的字符序列。 此类提供与StringBuffer的API，但不保证同步。 此类设计用作简易替换为StringBuffer在正在使用由单个线程字符串缓冲区的地方（如通常是这种情况）。 在可能的情况下，建议使用这个类别优先于StringBuffer ，因为它在大多数实现中将更快.

关于StringBuffer和StringBuilder的线程安全问题，可以分别查看类里面的方法，其中StringBuffer里面的方法都是用synchronized关键字进行了修饰，所以是线程安全的。也可以通过下面进行测试

启动10个线程，对字符串进行追加

```java
  public static void main(String[] args) throws Exception{
        StringBuilder stringBuilder=new StringBuilder();
        for(int i=0;i<10;i++){
            new Thread(new Runnable() {
                @Override
                public void run() {
                    for(int j=0;j<10000;j++){
                        stringBuilder.append("a");
                    }
                }
            }).start();
        }

        Thread.sleep(100);
        System.out.println("StringBuilder长度："+stringBuilder.length());
    }


```

最终的运行结果：StringBuilder长度：94301，这个与期望的值不太一样。

同理测试StringBuffer就不会存在上述的问题

```java
public static void main(String[] args) throws Exception{
    StringBuffer stringBuffer=new StringBuffer();
    for(int i=0;i<10;i++){
        new Thread(new Runnable() {
            @Override
            public void run() {
                for(int j=0;j<10000;j++){
                    stringBuffer.append("a");
                }
            }
        }).start();
    }

    Thread.sleep(100);
    System.out.println("StringBuilder长度："+stringBuffer.length());
}
```
关于Stringbuilder为什么会出现94301,而不是100000，可以通过源码分析得出

当StringBuilder调用append("a")这个方法时，调用如下：

```java
 @Override
    public StringBuilder append(String str) {
        super.append(str);
        return this;
    }
```

最终调用的是父类的方法

```java
public AbstractStringBuilder append(String str) {
    if (str == null)
        return appendNull();
    int len = str.length();
    ensureCapacityInternal(count + len);
    str.getChars(0, len, value, count);
    count += len;
    return this;
}
```
而在执行count+=len；这个动作时，如果存在多线程的情况下，就有可能导致 线程安全问题，所以才会出现上述的情况。