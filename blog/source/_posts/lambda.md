---
title: Java 8中的Lambda表达式
date: 2017-5-14 
categories: 并发编程
tags:
- java
---
2014年3月18日，Oracle公司发布了[Java SE 8](http://www.oracle.com/us/corporate/press/2172618)。距离Java 8的发布已经三年，最近正好抽空整理了Java 8的特性如下：

- 接口的默认方法
- Lambda 表达式
- 函数式接口
- 方法与构造函数引用
- Lambda 作用域
- 访问局部变量
- 访问对象字段与静态变量
- 访问接口的默认方法
- Date API
- Annotation 注解

本文将会重点讲解Java 8中的Lambda表达式，其他特性将会在后续文章中讲解。lambda 表达式，又被成为“闭包”或“匿名方法”。

## 背景
Java 是一门面向对象编程语言。面向对象编程语言和函数式编程语言中的基本元素（Basic Values）都可以动态封装程序行为：面向对象编程语言使用带有方法的对象封装行为，函数式编程语言使用函数封装行为。但这个相同点并不明显，因为Java 对象往往比较“重量级”：实例化一个类型往往会涉及不同的类，并需要初始化类里的字段和方法。

不过有些 Java 对象只是对单个函数的封装。例如下面这个典型用例：Java API 中定义了一个接口（一般被称为回调接口），用户通过提供这个接口的实例来传入指定行为。

```java
public interface ActionListener {
  void actionPerformed(ActionEvent e);
}
```
这里并不需要专门定义一个类来实现 ActionListener，因为它只会在调用处被使用一次。用户一般会使用匿名类型把行为内联（inline）：

```java
button.addActionListener(new ActionListener() {
  public void actionPerformed(ActionEvent e) {
    ui.dazzle(e.getModifiers());
  }
});
```
很多库都依赖于上面的模式。对于并行 API 更是如此，因为我们需要把待执行的代码提供给并行 API，并行编程是一个非常值得研究的领域，因为在这里摩尔定律得到了重生：尽管我们没有更快的 CPU 核心（core），但是我们有更多的 CPU 核心。而串行 API 就只能使用有限的计算能力。


### 匿名内部类
随着回调模式和函数式编程风格的日益流行，我们需要在Java中提供一种尽可能轻量级的将代码封装为数据（Model code as data）的方法。匿名内部类并不是一个好的 选择，因为：

- 语法过于冗余
- 匿名类中的 this 和变量名容易使人产生误解
- 类型载入和实例创建语义不够灵活
- 无法捕获非 final 的局部变量
- 无法对控制流进行抽象

### 函数式接口
尽管匿名内部类有着种种限制和问题，但是它有一个良好的特性，它和Java类型系统结合的十分紧密：每一个函数对象都对应一个接口类型。之所以说这个特性是良好的，是因为：

- 接口是 Java 类型系统的一部分
- 接口天然就拥有其运行时表示（Runtime representation）
- 接口可以通过 Javadoc 注释来表达一些非正式的协定（contract），例如，通过注释说明该操作应可交换（commutative）

接口只有一个方法，大多数回调接口都拥有这个特征：比如 Runnable 接口和 Comparator 接口。我们把这些只拥有一个方法的接口称为 函数式接口。（之前它们被称为 SAM类型，即 单抽象方法类型（Single Abstract Method））

我们并不需要额外的工作来声明一个接口是函数式接口：编译器会根据接口的结构自行判断（判断过程并非简单的对接口方法计数：一个接口可能冗余的定义了一个 Object 已经提供的方法，比如 toString()，或者定义了静态方法或默认方法，这些都不属于函数式接口方法的范畴）。不过API作者们可以通过 @FunctionalInterface 注解来显式指定一个接口是函数式接口（以避免无意声明了一个符合函数式标准的接口），加上这个注解之后，编译器就会验证该接口是否满足函数式接口的要求。

实现函数式类型的另一种方式是引入一个全新的 结构化 函数类型，我们也称其为“箭头”类型。例如，一个接收 String 和 Object 并返回 int 的函数类型可以被表示为 (String, Object) -> int。我们仔细考虑了这个方式，但出于下面的原因，最终将其否定：

- 它会为Java类型系统引入额外的复杂度，并带来 结构类型（Structural Type） 和 指名类型（Nominal Type） 的混用。（Java 几乎全部使用指名类型）
- 它会导致类库风格的分歧——一些类库会继续使用回调接口，而另一些类库会使用结构化函数类型
- 它的语法会变得十分笨拙，尤其在包含受检异常（checked exception）之后
- 每个函数类型很难拥有其运行时表示，这意味着开发者会受到 类型擦除（erasure） 的困扰和局限。比如说，我们无法对方法 m(T->U) 和 m(X->Y) 进行重载（Overload）

所以我们选择了“使用已知类型”这条路——因为现有的类库大量使用了函数式接口，通过沿用这种模式，我们使得现有类库能够直接使用 lambda 表达式。例如下面是 Java SE 7 中已经存在的函数式接口：

```java

java.lang.Runnable
java.util.concurrent.Callable
java.security.PrivilegedAction
java.util.Comparator
java.io.FileFilter
java.beans.PropertyChangeListener
```

除此之外，Java SE 8中增加了一个新的包：java.util.function，它里面包含了常用的函数式接口，例如：

```
Predicate<T>——接收 T 并返回 boolean
Consumer<T>——接收 T，不返回值
Function<T, R>——接收 T，返回 R
Supplier<T>——提供 T 对象（例如工厂），不接收值
UnaryOperator<T>——接收 T 对象，返回 T
BinaryOperator<T>——接收两个 T，返回 T
```

除了上面的这些基本的函数式接口，我们还提供了一些针对原始类型（Primitive type）的特化（Specialization）函数式接口，例如 IntSupplier 和 LongBinaryOperator。（我们只为 int、long 和 double 提供了特化函数式接口，如果需要使用其它原始类型则需要进行类型转换）同样的我们也提供了一些针对多个参数的函数式接口，例如 BiFunction<T, U, R>，它接收 T 对象和 U 对象，返回 R 对象。



### lambda表达式
匿名类型最大的问题就在于其冗余的语法。有人戏称匿名类型导致了“高度问题”。lambda表达式是匿名方法，它提供了轻量级的语法，从而解决了匿名内部类带来的“高度问题”。

```java
(int x, int y) -> x + y
() -> 42
(String s) -> { System.out.println(s); }
```

第一个 lambda 表达式接收 x 和 y 这两个整形参数并返回它们的和；第二个 lambda 表达式不接收参数，返回整数 ‘42’；第三个 lambda 表达式接收一个字符串并把它打印到控制台，不返回值。

lambda 表达式的语法由参数列表、箭头符号 -> 和函数体组成。函数体既可以是一个表达式，也可以是一个语句块：

- 表达式：表达式会被执行然后返回执行结果。
- 语句块：语句块中的语句会被依次执行，就像方法中的语句一样——
	- return 语句会把控制权交给匿名方法的调用者
	- break 和 continue 只能在循环中使用
	- 如果函数体有返回值，那么函数体内部的每一条路径都必须返回值

表达式函数体适合小型 lambda 表达式，它消除了 return 关键字，使得语法更加简洁。

lambda 表达式也会经常出现在嵌套环境中，比如说作为方法的参数。为了使 lambda 表达式在这些场景下尽可能简洁，我们去除了不必要的分隔符。不过在某些情况下我们也可以把它分为多行，然后用括号包起来，就像其它普通表达式一样。

## 实战应用

### Function

Function 接口有一个参数并且返回一个结果，并附带了一些可以和其他函数组合的默认方法：`andThen`和`compose`。

```java
    @Test
    public void testFun() {
        //Function 接口有一个参数并且返回一个结果
        Function<String, Integer> toInteger = (t) -> Integer.valueOf(t);
        System.out.println("compose: " + toInteger.andThen(a -> a + 10).compose(str -> str + "1").apply("123"));

        Function<String, String> backToString = toInteger.andThen(String::valueOf);
        Function<String, Integer> f = toInteger.compose(backToString);
        int str = f.apply("123");
        System.out.println(str);
    }
```
`compose`和`andThen`中定义的Function应用顺序正好相反，首先应用compose中的方法，其次才会应用当前Function。

### Supplier
Supplier 接口返回一个任意范型的值，和Function接口不同的是该接口没有任何参数。代码如下:

```java
    @Test
    public void testSupplier() {
        //Supplier 接口返回一个任意范型的值，和Function接口不同的是该接口没有任何参数
        Supplier sp = () -> "sp";
        System.out.println(sp.get());
    }
```
如上代码将会返回一个字符串“sp”，通过`get`方法获取到返回的值。

### Predicate
Predicate 接口只有一个参数，返回boolean类型。该接口包含多种默认方法来将Predicate组合成其他复杂的逻辑（比如：与，或，非）：

```java
    @Test
    public void testPredicate() {
        Predicate<String> isEmpty = String::isEmpty;
        Predicate<String> isNotEmpty = isEmpty.negate();
        isEmpty.and(str -> str.equals("test"));
        System.out.println("tes: " + isEmpty.and(str -> str.equals("test")).test("tes"));

    }
```
如上代码判断了字符串是否为空，并应用了与、非操作。

### Consumer
Consumer 接口表示执行在单个参数上的操作，主要的方法为`andThen`和`accept`。

```java
    @Test
    public void testConsumer() {
        SimpleDateFormat df = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");//设置日期格式
        Consumer<String> greeter = (p) -> System.out.println("Hello, " + p);
        greeter.andThen((t) -> System.out.println("now is :" + df.format(new Date()))).accept("Skywalker");

    }
```
`accept`表示接收指定的参数执行操作，`andThen`表示当前操作结束之后附加的操作。
### Comparator
`Comparator`接口用于比较， Java 8在此之上添加了多种默认方法，如`reversed`和`thenComparing`等。

```java
    @Test
    public void testComparator() {
        Comparator<String> comparator = String::compareTo;

        String str1 = "eeeabc";

        String str2 = "bcd";

        System.out.println("str比较大小：" + comparator.compare(str1, str2));

        System.out.println("str比较大小反转：" + comparator.reversed().compare(str1, str2));

    }
```
### Optional
用来防止NullPointerException异常的辅助类型，现在看看这个接口能干什么：

`Optional`被定义为一个简单的容器，其值可能是null或者不是null。在Java 8之前一般某个函数应该返回非空对象但是偶尔却可能返回了null，而在Java 8中，不推荐你返回null而是返回Optional。

```java
    @Test
    public void testOptional() {
        //用来防止NullPointerException异常的辅助类型
        List<String> list = Arrays.asList("ab", "bc");
        System.out.println(list.stream().findFirst().orElse("null str"));
        Optional<String> optional = Optional.of("hello");

        optional.isPresent();           // true
        optional.get();                 // "hello"
        optional.orElse("hi");    // "hello"

        optional.ifPresent((s) -> System.out.println("字符串不为空：" + s));
    }
```
`optional.orElse`用来对异常情况返回预设的返回结果。
### Stream
java.util.Stream 表示能应用在一组元素上一次执行的操作序列。Stream 操作分为中间操作或者最终操作两种，最终操作返回一特定类型的计算结果，而中间操作返回Stream本身，这样你就可以将多个操作依次串起来。Stream 的创建需要指定一个数据源，比如 java.util.Collection的子类，List或者Set， Map不支持。

```java
    @Test
    public void testSort() {
        List<String> list = Arrays.asList("abe", "abc");

        list = list.stream().filter(s -> s.startsWith("a")).sorted().collect(Collectors.toList());
        list.stream().forEach(System.out::println);
    }
```

### Map
中间操作map会将元素根据指定的Function接口来依次将元素转成另外的对象，下面的示例展示了将字符串转换为大写字符串。你也可以通过map来讲对象转换成其他类型，map返回的Stream类型是根据你map传递进去的函数的返回值决定的。

```java
    @Test
    public void testMap() {
        List<String> list = Arrays.asList("abe", "abc");
        //map返回的Stream类型是根据传递进去的函数的返回值决定
        list.stream().map(String::toCharArray).forEach(array -> System.out.println(array.length));
    }
```

### Match
Stream提供了多种匹配操作，允许检测指定的Predicate是否匹配整个Stream。所有的匹配操作都是最终操作，并返回一个boolean类型的值。

```java
    @Test
    public void testMatch() {
        List<String> list = Arrays.asList("ab", "abc");

        boolean anyMatch = list.stream().map(String::toCharArray).anyMatch(array -> array.length == 3);

        boolean allMatch = list.stream().map(String::toCharArray).allMatch(array -> array.length == 3);

        boolean noneMatch = list.stream().map(String::toCharArray).noneMatch(array -> array.length == 3);

        System.out.println("anyMatch:" + anyMatch);
        System.out.println("allMatch:" + allMatch);
        System.out.println("noneMatch:" + noneMatch);
    }
```

### Reduce
这是一个最终操作，允许通过指定的函数来讲stream中的多个元素规约为一个元素，规越后的结果是通过Optional接口表示的：

```java
    @Test
    public void testReduce() {
        List<String> list = Arrays.asList("ab", "abc", "abcd");
        Optional<String> reduce = list.stream().reduce((s1, s2) -> s1 + ":" + s2);
        reduce.ifPresent(s -> System.out.println(s));
    }
```
### ParallelStream
串行Stream上的操作是在一个线程中依次完成，而并行Stream则是在多个线程上同时执行。

```java
    @Test
    public void testParallelStream() {
        int max = 1000000;
        List<String> values = new ArrayList<>(max);
        for (int i = 0; i < max; i++) {
            UUID uuid = UUID.randomUUID();
            values.add(uuid.toString());
        }

        long t0 = System.nanoTime();

        long count = values.parallelStream().sorted().count();
        System.out.println(count);

        long t1 = System.nanoTime();

        long millis = TimeUnit.NANOSECONDS.toMillis(t1 - t0);
        System.out.println(String.format("sequential sort took: %d ms", millis));

    }
```
如上所示为并行排序，排序这个Stream耗时明显低于串行。
### Map方法
Map类型不支持stream，不过Map提供了一些新的有用的方法来处理一些日常任务。

```java
    @Test
    public void testMapFun() {
        Map<Integer, String> map = new HashMap<>();

        for (int i = 0; i < 10; i++) {
            map.putIfAbsent(i, "val" + i);
        }

        map.forEach((id, val) -> System.out.println(val));

        map.computeIfPresent(3, (num, val) -> val + num);
        System.out.println(map.get(3));

        map.computeIfPresent(9, (num, val) -> null);
        System.out.println(map.containsKey(9));

        map.computeIfAbsent(23, num -> "val" + num);
        System.out.println(map.get(23));

        map.putIfAbsent(3, "bam");
        System.out.println(map.get(3));

        map.remove(3, "val3");
        System.out.println(map.get(3));

        //Merge时，如果键名不存在则插入，否则则对原键对应的值做合并操作并重新插入到map中
        map.merge(9, "val9", (value, newValue) -> value.concat(newValue));
        System.out.println(map.get(9));

        map.merge(9, "concat", (value, newValue) -> value.concat(newValue));
        System.out.println(map.get(9));

    }
```
### UnaryOperator
继承自`Function`接口，表示对单个操作数的操作，该操作生成与其操作数相同类型的结果。

```java
    @Test
    public void testUnaryOperator() {
        UnaryOperator<String> unaryOperator = str -> str + "-test";

        System.out.println(unaryOperator.apply("123"));
    }
```

## 小结
本文主要介绍了Java8中的Lambda表达式，选择其中常用的方法进行了简单的应用讲解。Lambda表达式是Java SE 8中一个重要的新特性。Lambda表达式允许你通过表达式来代替功能接口。Lambda表达式就和方法一样，它提供了一个正常的参数列表和一个使用这些参数的主体(body,可以是一个表达式或一个代码块)。
Lambda表达式还增强了集合库，包括java.util.function 包以及java.util.stream包。Lambda表达式非常简洁，大大简化代码行数，使代码在一定程度上变的简洁干净，但是同样的，这可能也会是一个缺点，由于省略了太多东西，代码可读性有可能在一定程度上会降低，这个完全取决于你使用lambda表达式的位置所设计的API是否被你的代码的其他阅读者所熟悉。

### 参考文档
1. [Java 8中一些常用的全新的函数式接口](https://www.cnblogs.com/heimianshusheng/p/5672641.html)
2. [深入理解Java 8 Lambda（语言篇——lambda，方法引用，目标类型和默认方法）](http://lucida.me/blog/java-8-lambdas-insideout-language-features/)


