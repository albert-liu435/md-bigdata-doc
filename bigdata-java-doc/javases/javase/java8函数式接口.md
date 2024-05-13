java8函数式接口 

### Supplier

函数式接口（Functional Interface）是JDK 8中新增的特性，其实也是lambda表达式编程模式中的一个很重要的构成。我们先看看什么是函数式接口。
函数式接口：有且只有一个抽象方法的接口，为函数式接口。除此限制之外，函数式接口仍然遵循接口的其他基本设计原则，比如允许声明static属性、static方法，也允许有默认方法等
	
Supplier中文翻译就是供应商,对应到java中就是用来提供结果的,其功能类似一个工厂,可以不断的创建对象
Supplier里面只有一个 T get()方法。

其简单用法如下：

```java
//用来获取两个User对象    
Supplier<User> supplier = User::new;
    User user1 = supplier.get();
    user1.setAge(12);
    user1.setName("zhangsan");
    System.out.println(user1);
    System.out.println(supplier.get());
    System.out.println(supplier.get());
```
另一种写法如下：

//返回两个不同的结果

```java
        Supplier<User> supplier1 = () -> new User();
        System.out.println(supplier1.get());
        System.out.println(supplier1.get());
```

```java
    //表示用来返回两个Book对象
    User user = new User();
    Supplier<Book> supplier2 = user::supplier;
    System.out.println(supplier2.get().toString());
    System.out.println(supplier2.get().toString());
```

同理还有DoubleSupplier，IntSupplier，LongSupplier，函数式接口。

源码见[github](https://github.com/albert-liu435/rookies-javases/tree/master/rookie-javase)

### Consumer

consumer接口就是一个消费型的接口，通过传入参数.其中里面只定义了两个方法

```java
@FunctionalInterface
public interface Consumer<T> {

    /**
     * 根据给定的参数执行定义的操作
     *
     * @param t the input argument
     */
    void accept(T t);

    /**
     * 根据给定的参数执行定义的操作，然后再执行after定义的操作
     */
    default Consumer<T> andThen(Consumer<? super T> after) {
        Objects.requireNonNull(after);
        return (T t) -> { accept(t); after.accept(t); };
    }
}
```

Consumer简单的用法如下：

```java
//lambda表达式实例
Consumer<String> consumer = (s) -> {
    System.out.println(s);
};
consumer.accept("hello consumer");

//接口实现方法
Consumer<String> stringConsumer = new Consumer<String>() {
    @Override
    public void accept(String s) {
        System.out.println(s);
    }
};

stringConsumer.accept("hello impl Consumer");


Consumer<Integer> consumer1 = integer -> System.out.println(integer + 2);

Consumer<Integer> consumer2 = integer -> System.out.println(integer + 3);

Consumer<Integer> integerConsumer = consumer2.andThen(consumer1);
//先打印出6,后再打印出5
integerConsumer.accept(3);
```

同理还有：BiConsumer，DoubleConsumer，IntConsumer，LongConsumer，ObjIntConsumer，ObjLongConsumer

### Function

function为接受一个输入参数T，返回一个结果R

下面看一下里面定义的方法

```java
@FunctionalInterface
public interface Function<T, R> {

    /**
     * 接受一个参数，执行指定的操作后返回一个结果
     */
    R apply(T t);

    /**
     * 返回一个 先执行before函数对象apply方法，再执行当前函数对象apply方法的 函数对象
     */
    default <V> Function<V, R> compose(Function<? super V, ? extends T> before) {
        Objects.requireNonNull(before);
        return (V v) -> apply(before.apply(v));
    }

    /**
     *刚好跟compose相反，返回一个 先执行当前函数对象apply方法， 再执行after函数对象apply方法的 函数对象。
     */
    default <V> Function<T, V> andThen(Function<? super R, ? extends V> after) {
        Objects.requireNonNull(after);
        return (T t) -> after.apply(apply(t));
    }

    /**
     * 返回一个执行了apply()方法之后只会返回输入参数的函数对象。该方法代表一个lambda表达式
     */
    static <T> Function<T, T> identity() {
        return t -> t;
    }
}
```

demo实例

```java

    @Test
    public void apply(){

        //定义一个实现类
        Function<Integer,Integer> function= new Function<Integer, Integer>() {
            @Override
            public Integer apply(Integer integer) {

                return integer+5;
            }
        };

        Integer apply = function.apply(3);
        System.out.println(apply);

        //采用lambda表达式的写法
        Function<Integer,Integer> function1=integer -> integer+3;
        System.out.println(function1.apply(3));

        Function<Integer,Integer> function2=integer -> {
            return integer+3;
        };

        System.out.println(function2.apply(5));



        MyFunction<String,String> myFunction=new MyFunction<>();
        String hello = myFunction.getProcessor().apply("hello");
        System.out.println(hello);


    }


    @Test
    public void  compose(){

        Function<Integer,Integer> function= new Function<Integer, Integer>() {
            @Override
            public Integer apply(Integer integer) {
                //这里会打印出2
                System.out.println(integer);
                return integer+5;
            }
        };

        Function<Integer,Integer> function1=integer -> integer+2;
        Function<Integer, Integer> compose = function1.compose(function);
        Integer apply = compose.apply(2);
        System.out.println(apply);

    }


    @Test
    public void andThen(){
        Function<Integer,Integer> function= new Function<Integer, Integer>() {
            @Override
            public Integer apply(Integer integer) {
                //这里会打印出4
                System.out.println(integer);
                return integer+5;
            }
        };

        Function<Integer,Integer> function1=integer -> integer+2;
        Function<Integer, Integer> compose = function1.andThen(function);
        Integer apply = compose.apply(2);
        System.out.println(apply);
    }


    @Test
    public void identity(){

        Stream<String> stream = Stream.of("This", "is", "a", "test");
        Map<String, Integer> map = stream.collect(Collectors.toMap(Function.identity(), String::length));
        System.out.println(map);
        Function<Integer,Integer> function1=Function.identity();
        Function<Integer,Integer> function2=t->t;
        Integer apply = function1.apply(2);
        System.out.println(apply);

        Integer apply2 = function2.apply(3);
        System.out.println(apply2);


    }
```

同理还有：BiFunction<T, U, R>，DoubleFunction<R>，DoubleToIntFunction，DoubleToLongFunction，IntFunction<R>，IntToDoubleFunction，IntToLongFunction，LongFunction<R>，LongToDoubleFunction，LongToIntFunction，ToDoubleBiFunction<T, U>，ToDoubleFunction<T>，ToIntBiFunction<T, U>，ToIntFunction<T>，ToLongBiFunction<T, U>，ToLongFunction<T>等用法基本类似

### Predicate

prediccate<T>,接受一个参数，主要用于逻辑判断

```java
@FunctionalInterface
public interface Predicate<T> {

    /**
     * 进行逻辑判断
     */
    boolean test(T t);

    /**
     * 返回一个限制性test的逻辑判断，然后在执行other的逻辑判断的函数对象，当两个执行的时候都返回ture时，才会返回true.
     */
    default Predicate<T> and(Predicate<? super T> other) {
        Objects.requireNonNull(other);
        return (t) -> test(t) && other.test(t);
    }

    /**
     * 返回一个lambda表达式，类似Function函数中的identity()方法的写法
     */
    default Predicate<T> negate() {
        return (t) -> !test(t);
    }

    /**
     * 返回一个限制性test的逻辑判断，然后在执行other的逻辑判断的函数对象，当两个执行的时候其中一个回ture时，就会返回true.
     */
    default Predicate<T> or(Predicate<? super T> other) {
        Objects.requireNonNull(other);
        return (t) -> test(t) || other.test(t);
    }

    /**
     * 该方法接收一个Object对象,返回一个Predicate类型.此方法用于判断第一个test的方法与第二个test方法相同(equal).
     */
    static <T> Predicate<T> isEqual(Object targetRef) {
        return (null == targetRef)
                ? Objects::isNull
                : object -> targetRef.equals(object);
    }
}
```

demo实例

```java

    @Test
    public void test() {


        //定义实现类的写法
        Predicate<String> predicate = new Predicate<String>() {
            @Override
            public boolean test(String s) {
                return s.equals("a");
            }
        };

//        //lambda表达式写法
//        Predicate<String> predicate=x->{
//            return x.equals("a");
//        };

        //返回true
        boolean a = predicate.test("a");
        System.out.println(a);
        //返回false
        boolean b = predicate.test("b");
        System.out.println(b);

    }


    @Test
    public void and() {


        Predicate<String> predicate1=new Predicate<String>() {
            @Override
            public boolean test(String s) {
                boolean a = s.equals("a");
                //返回true
                System.out.println(a);
                return a;
            }
        };


        Predicate<String> predicate2=new Predicate<String>() {
            @Override
            public boolean test(String s) {
                //返回false
                boolean b = s.equals("b");
                System.out.println(b);
                return b;
            }
        };

        Predicate<String> and = predicate1.and(predicate2);
        //返回false
        boolean all = and.test("a");
        System.out.println(all);


    }

    @Test
    public void or(){

        Predicate<String> predicate1=new Predicate<String>() {
            @Override
            public boolean test(String s) {
                boolean a = s.equals("a");
                //返回true
                System.out.println(a);
                return a;
            }
        };


        Predicate<String> predicate2=new Predicate<String>() {
            @Override
            public boolean test(String s) {
                //返回false
                boolean b = s.equals("b");
                System.out.println(b);
                return b;
            }
        };

        //下面这两种写法最终返回的结果一致，但是执行过程略有差异
       // Predicate<String> and = predicate1.or(predicate2);

        Predicate<String> and = predicate2.or(predicate1);

        //返回true
        boolean all = and.test("a");
        System.out.println(all);




    }


    @Test
    public void isEqual() {

        //返回true
        Predicate<Object> test = Predicate.isEqual("test");
        boolean test1 = test.test("test");
        System.out.println(test1);

        //返回false
        boolean test2 = Predicate.isEqual("hello").test("test");
        System.out.println(test2);


    }
```

同理还有BiPredicate<T, U>，DoublePredicate，IntPredicate，LongPredicate等函数用法类似。
