java8 Stream

Java 8 API添加了一个新的抽象称为流Stream，可以让你以一种声明的方式处理数据。Stream 使用一种类似用 SQL 语句从数据库查询数据的直观方式来提供一种对 Java 集合运算和表达的高阶抽象。Stream API可以极大提高Java程序员的生产力，让程序员写出高效率、干净、简洁的代码。这种风格将要处理的元素集合看作一种流， 流在管道中传输， 并且可以在管道的节点上进行处理， 比如筛选， 排序，聚合等。元素流在管道中经过中间操作（intermediate operation）的处理，最后由最终操作(terminal operation)得到前面处理的结果。

#### Stream具有如下特点

stream不存储数据，而是按照特定的规则对数据进行计算，一般会输出结果。

stream不会改变数据源，通常情况下会产生一个新的集合或一个值。

stream具有延迟执行特性，只有调用终端操作时，中间操作才会执行。



#### Stream接口方法

```java

public interface Stream<T> extends BaseStream<T, Stream<T>> {

    /**
     * filter 接受一个Predicate函数，该方法用于通过设置的条件过滤出元素filter 方法用于通过设置的条件过滤出元素,
     */
    Stream<T> filter(Predicate<? super T> predicate);

    /**
     * 接受一个Function函数，相当于按照某种方法操作后，然后再返回
     */
    <R> Stream<R> map(Function<? super T, ? extends R> mapper);

    /**
     * 接受一个ToIntFunction,相当于将里面的数据转换为int后再返回
     */
    IntStream mapToInt(ToIntFunction<? super T> mapper);

 
    LongStream mapToLong(ToLongFunction<? super T> mapper);


    DoubleStream mapToDouble(ToDoubleFunction<? super T> mapper);

    /**
     * 可以认为将多个流转换为一个流的中间操作
     */
    <R> Stream<R> flatMap(Function<? super T, ? extends Stream<? extends R>> mapper);

    IntStream flatMapToInt(Function<? super T, ? extends IntStream> mapper);

    LongStream flatMapToLong(Function<? super T, ? extends LongStream> mapper);


    DoubleStream flatMapToDouble(Function<? super T, ? extends DoubleStream> mapper);

    /**
     * 去重操作，对重复元素去重，底层使用了equals方法
     */
    Stream<T> distinct();

    /**
     * 排序操作，按照默认的排序规则进行排序
     */
    Stream<T> sorted();

    /**
     *排序操作，按照指定的比较器进行排序
     */
    Stream<T> sorted(Comparator<? super T> comparator);

    /**
     * 如同于map，能得到流中的每一个元素。但map接收的是一个Function表达式，有返回值；而peek接收的是Consumer表达式，没有返回值。
     */
    Stream<T> peek(Consumer<? super T> action);

    /**
    * 遍历迭代
    */
    
    void forEach(Consumer<? super T> action);
 
    void forEachOrdered(Consumer<? super T> action);

    
    /**
    * 转换为数组
    */
        Object[] toArray();
     <A> A[] toArray(IntFunction<A[]> generator);
    
    /**
     * reduce 规约操作，将整个数据流的值规约为一个值，count、min、max底层就是使用reduce
     */
    T reduce(T identity, BinaryOperator<T> accumulator);


    /**
     * 第一次执行时，accumulator函数的第一个参数为流中的第一个元素，第二个参数为流中元素的第二个元素；第二次执行时，第一个参数为第一次函数执行的结果，第二个参数为流中的第三个元素；依次类推
     */
    Optional<T> reduce(BinaryOperator<T> accumulator);

    /**
     * 在串行流(stream)中，该方法跟reduce(BinaryOperator<T> accumulator)方法一样，即第三个参数combiner不会起作用。在并行流(parallelStream)中,我们知道流被fork join出多个线程进行执行，此时每个线程的执行流程就跟第二个方法reduce(identity,accumulator)一样，而第三个参数combiner函数，则是将每个线程的执行结果当成一个新的流，然后使用第一个方法reduce(accumulator)流程进行规约。
     * 
     */
    <U> U reduce(U identity,
                 BiFunction<U, ? super T, U> accumulator,
                 BinaryOperator<U> combiner);

    /**
     * 将流转换为其他形式,接受一个Collector接口的实现,用于给Stream中的元素做汇总的方法
     */
    <R> R collect(Supplier<R> supplier,
                  BiConsumer<R, ? super T> accumulator,
                  BiConsumer<R, R> combiner);

    /**
     *将流转换为其他形式,接受一个Collector接口的实现,用于给Stream中的元素做汇总的方法
     */
    <R, A> R collect(Collector<? super T, A, R> collector);

    /**
     * 返回流中的最小元素
     */
    Optional<T> min(Comparator<? super T> comparator);

    /**
     * 返回流中的最大元素
     */
    Optional<T> max(Comparator<? super T> comparator);

    /**
     * 返回流中的元素总个数
     */
    long count();

    /**
     * 接收一个 Predicate 函数，当流中每个元素都符合该断言时才返回true，否则返回false
     */
    boolean anyMatch(Predicate<? super T> predicate);

    /**
     * 接收一个 Predicate 函数，只要流中有一个元素满足该断言则返回true，否则返回false
     */
    boolean allMatch(Predicate<? super T> predicate);

    /**
     * 接收一个 Predicate 函数，当流中每个元素都不符合该断言时才返回true，否则返回false
     */
    boolean noneMatch(Predicate<? super T> predicate);

    /**
     * 返回流中第一个元素
     */
    Optional<T> findFirst();

    /**
     *返回流中的任意元素
     */
    Optional<T> findAny();

    // Static factories

    /**
     */
    public static<T> Builder<T> builder() {
        return new Streams.StreamBuilderImpl<>();
    }

    /**
     */
    public static<T> Stream<T> empty() {
        return StreamSupport.stream(Spliterators.<T>emptySpliterator(), false);
    }

    /**
     */
    public static<T> Stream<T> of(T t) {
        return StreamSupport.stream(new Streams.StreamBuilderImpl<>(t), false);
    }

    /**
     */
    @SafeVarargs
    @SuppressWarnings("varargs") // Creating a stream from an array is safe
    public static<T> Stream<T> of(T... values) {
        return Arrays.stream(values);
    }

    /**
     */
    public static<T> Stream<T> iterate(final T seed, final UnaryOperator<T> f) {
        Objects.requireNonNull(f);
        final Iterator<T> iterator = new Iterator<T>() {
            @SuppressWarnings("unchecked")
            T t = (T) Streams.NONE;

            @Override
            public boolean hasNext() {
                return true;
            }

            @Override
            public T next() {
                return t = (t == Streams.NONE) ? seed : f.apply(t);
            }
        };
        return StreamSupport.stream(Spliterators.spliteratorUnknownSize(
                iterator,
                Spliterator.ORDERED | Spliterator.IMMUTABLE), false);
    }

    /**

     */
    public static<T> Stream<T> generate(Supplier<T> s) {
        Objects.requireNonNull(s);
        return StreamSupport.stream(
                new StreamSpliterators.InfiniteSupplyingSpliterator.OfRef<>(Long.MAX_VALUE, s), false);
    }

    /**

     */
    public static <T> Stream<T> concat(Stream<? extends T> a, Stream<? extends T> b) {
        Objects.requireNonNull(a);
        Objects.requireNonNull(b);

        @SuppressWarnings("unchecked")
        Spliterator<T> split = new Streams.ConcatSpliterator.OfRef<>(
                (Spliterator<T>) a.spliterator(), (Spliterator<T>) b.spliterator());
        Stream<T> stream = StreamSupport.stream(split, a.isParallel() || b.isParallel());
        return stream.onClose(Streams.composedClose(a, b));
    }

    /**

     */
    public interface Builder<T> extends Consumer<T> {

        /**

         */
        @Override
        void accept(T t);

        /**

         */
        default Builder<T> add(T t) {
            accept(t);
            return this;
        }

        /**

         */
        Stream<T> build();

    }
}

```

上面的方法可以分类成如下：

![20181223012834784](.\pic\20181223012834784.png)

无状态：指元素的处理不受之前元素的影响；

有状态：指该操作只有拿到所有元素之后才能继续下去。

非短路操作：指必须处理所有元素才能得到最终结果；

短路操作：指遇到某些符合条件的元素就可以得到最终结果，如 A || B，只要A为true，则无需判断B的结果。

#### Stream的具体用法demo

```java

    @Test
    public void filter() {

        List<String> strings = Arrays.asList("a", "ab", "abc", "abcd", "abcde", "abcdef", "abcdefg", "abcdefgh");
        //过滤出字符串长度大于3的字符串
        long count = strings.stream().filter(s -> {
            return s.length() > 3;
        }).count();

        strings.stream().filter(s -> {
            return s.length() > 3;
        }).forEach(
                System.out::println
        );

        System.out.println(count);
        //System.out.println(s1);
    }

    @Test
    public void map() {

        //将字符串添加"h"再过滤出大于3的字符串
        List<String> strings = Arrays.asList("a", "ab", "abc", "abcd", "abcde", "abcdef", "abcdefg", "abcdefgh");
        long count = strings.stream().map(s -> s + "h").filter(s -> s.length() > 3).count();
        System.out.println(count);

        List<String> collect = strings.stream().map(s -> s + "h").filter(s -> s.length() > 3).collect(Collectors.toList());
        for (String s : collect) {
            System.out.println(s);
        }

    }

    @Test
    public void mapToInt() {

        ToIntFunction<String> toIntFunction = new ToIntFunction<String>() {
            @Override
            public int applyAsInt(String value) {
                return new Integer(value);
            }
        };


        //将字符串数字转换为int类型
        //mapToLong mapToDouble功能类似
        List<String> strings = Arrays.asList("1", "2", "3", "4", "5", "6", "7", "8");

        int[] ints = strings.stream().mapToInt(s -> new Integer(s)).toArray();
        System.out.println(ints);
        for (int anInt : ints) {
            System.out.println(anInt);
        }


    }

    @Test
    public void flatMap() throws Exception {

        Stream<String> lines = Files.lines(Paths.get("C:\\work\\IDEAWorkSpace\\rookie-project\\rookies-javases\\rookie-javase\\src\\main\\java\\com\\rookie\\bigdata\\util\\stream\\StreamTestMain.java"), StandardCharsets.UTF_8);
        //将数据按照空格进行分割
        Stream<String> words = lines.flatMap(line -> Stream.of(line.split("")));
        words.forEach(
                System.out::println
        );

        //将两个list合并成一个list
        List<Integer> list = (List<Integer>) Stream.of(Arrays.asList(1, 2, 3, 4, 5, 6), Arrays.asList(8, 9, 10, 11, 12))
                .flatMap(list1 -> list1.stream()).collect(Collectors.toList());

        for (int i = 0, length = list.size(); i < length; i++) {
            System.out.println(list.get(i));
        }


        //扁平化流
        //找出数组中唯一的字符
        String[] strArray = {"hello", "world"};

        //具体实现
        List<String> res = Arrays.stream(strArray)
                .map(w -> w.split(""))
                .flatMap(Arrays::stream)
                .distinct() //进行去重操作
                .collect(Collectors.toList());
        System.out.println(res);


        //flatMapToInt,flatMapToLong,flatMapToDouble功能类似

    }

    @Test
    public void distinct() {


        Stream<String> stream = Stream.of("I", "love", "write", "java", "code", "I", "love", "write", "java", "code");
        //进行去重操作,最后只剩下"I", "love", "write", "java", "code"
        stream.distinct()
                .forEach(str -> System.out.println(str));

    }

    @Test
    public void sorted() {

        //进行排序操作
        Stream<String> stream = Stream.of("I", "love", "write", "java", "code", "I", "love", "write", "java", "code");
        //进行去重操作,最后只剩下"I", "love", "write", "java", "code"
        stream.distinct().sorted().forEach(
                System.out::println
        );
        System.out.println("----------------------------");

        //按照数字长度进行倒叙排列
        Stream.of("I", "love", "write", "java", "code", "I", "love", "write", "java", "code").distinct().sorted((s1, s2) -> {
            return s2.length() - s1.length();
        }).forEach(
                System.out::println
        );


    }


    @Test
    public void peek() {
        List<String> collect = Stream.of("one", "two", "three", "four")
                .filter(e -> e.length() > 3)
                .peek(e -> System.out.println("Filtered value: " + e))
                .map(String::toUpperCase)
                .peek(e -> System.out.println("Mapped value: " + e))
                .collect(Collectors.toList());

        System.out.println(collect.size());


        Stream<String> stream = Stream.of("hello", "Stream");
        stream.peek(System.out::println).collect(Collectors.toList());
    }

    @Test
    public void limit() {
        //限流操作，比如数据流中有4个 我只要出前2个就可以使用。
        Stream<String> stream = Stream.of("one", "two", "three", "four");
        List<String> collect = stream.limit(2).collect(Collectors.toList());
        System.out.println(collect.get(0));


    }

    @Test
    public void skip() {
        //跳过操作，跳过某些元素
        Stream<String> stream = Stream.of("one", "two", "three", "four");
        List<String> collect = stream.skip(2).collect(Collectors.toList());

        for (String s : collect) {
            System.out.println(s);
        }

    }


    @Test
    public void reduce() {

        //对所有的数据进行求和
        List<Integer> list = Arrays.asList(2, 3, 1, 10, 23, 18);
        // Stream<Integer> integerStream = Stream.of(2, 3, 1, 10, 23, 18);

        Optional<Integer> reduce = list.stream().reduce((a, b) -> a + b);

        Integer integer = reduce.get();
        System.out.println(integer);
        //在10的基础上对所有的数据求和
        Integer reduce1 = list.stream().reduce(10, Integer::sum);
        System.out.println(reduce1);

        Integer reduce2 = list.stream().reduce(10, (a, b) -> a + b);
        System.out.println(reduce2);

        System.out.println("-------------------------------------------------");

    //这里运用的是串行流
        Integer singleStream  = list.stream().reduce(0,
                (a, b) -> {
                    System.out.println("accumulator: a:" + a + "  b:" + b);
                    return a + b;
                },
                (a, b) -> {
                    System.out.println("combiner: a:" + a + "  b:" + b);
                    return a * b;
                });
        System.out.println(singleStream);

        //这里运用的并行流
        Integer parallelStream  = list.parallelStream().reduce(0,
                (a, b) -> {
                    System.out.println(Thread.currentThread().getName());
                    System.out.println("accumulator: a:" + a + "  b:" + b);
                    return a + b;
                },
                (a, b) -> {
                    System.out.println(Thread.currentThread().getName());
                    System.out.println("combiner: a:" + a + "  b:" + b);
                    return a * b;
                });
        System.out.println(parallelStream);


    }

    @Test
    public void reduce2(){

        List<Integer> list = Arrays.asList(1, 2, 3, 4, 5, 6);


        //这里运用的并行流,最后算出集合中数据的积
        Integer parallelStream  = list.parallelStream().reduce(0,
                (a, b) -> {
                   // System.out.println();
                    System.out.println(Thread.currentThread().getName()+" accumulator: a:" + a + "  b:" + b);
                    return a + b;
                },
                (a, b) -> {
                   // System.out.println(Thread.currentThread().getName());
                    System.out.println(Thread.currentThread().getName()+" combiner: a:" + a + "  b:" + b);
                    return a * b;
                });
        System.out.println(parallelStream);
    }



    @Test
    public void conllect(){


        //将流转换为其他形式,接受一个Collector接口的实现,用于给Stream中的元素做汇总的方法
        Stream<String> stringStream = Stream.of("hello","everyone","I","love","java");
        List<String> asList = stringStream.collect(Collectors.toList());

        for (String s : asList) {
            System.out.println(s);
        }

        Stream<String> stringStream2 = Stream.of("hello","everyone","I","love","java");
        String concat = stringStream2.collect(StringBuilder::new, StringBuilder::append,
                StringBuilder::append).toString();
        System.out.println(concat);

    }


    @Test
    public void min(){
        //取出最小值
        List<Integer> list = Arrays.asList(1, 2, 3, 4, 5);

        Optional<Integer> min = list.stream().min(Integer::compareTo);
        System.out.println(min.get());

        Optional<Integer> min1 = list.stream().min((a,b)->{
            return (a <= b) ? -1 : 1;
        });
        System.out.println(min1.get());

    }

    @Test
    public void max(){
        //取出最小值
        List<Integer> list = Arrays.asList(1, 2, 3, 4, 5);

        Optional<Integer> min = list.stream().max(Integer::compareTo);
        System.out.println(min.get());
        
        boolean allMatch = list.stream().allMatch(e -> e > 10);
        boolean noneMatch = list.stream().noneMatch(e -> e > 10);
        boolean anyMatch = list.stream().anyMatch(e -> e > 4);

        Integer findFirst = list.stream().findFirst().get();
        Integer findAny = list.stream().findAny().get();

        long count = list.stream().count();
    }
```

参考：[Java8 Stream：2万字20个实例，玩转集合的筛选、归约、分组、聚合](https://blog.csdn.net/mu_wind/article/details/109516995)

[Java 8 stream的详细用法](https://blog.csdn.net/y_k_y/article/details/84633001)

