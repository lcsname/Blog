---
layout: posts
title: 一个可以让编程更加快乐的工具---Google Guava
date: 2018-08-10 00:38:36
categories: 工具类
tags: [工具类]
top: false
---
Guava 工程包含了若干被Google的 Java项目广泛依赖 的核心库，例如：集合 [collections] 、缓存 [caching] 、原生类型支持 [primitives support] 、并发库 [concurrency libraries] 、通用注解 [common annotations] 、字符串处理 [string processing] 、I/O 等等。 所有这些工具每天都在被Google的工程师应用在产品服务中。作为Java程序员，使用Guava库来减少项目中大量的样板代码。目前Google Guava在实际应用中非常广泛，学习使用Google Guava可以是编程更加快乐，可以写出更加优雅的JAVA代码！它的中文教程：[中文教程](https://www.ctolib.com/docs-google-guava-official-tutorial-c-index.html)，有不会的时候就来就进来查查喽。

<!--more--> 

#### 一、以面向对象思想处理字符串:Joiner/Splitter/CharMatcher

> JDK提供的String还不够好么?
>
> 举个栗子，比如String提供的split方法，我们得关心空字符串吧，还得考虑返回的结果中存在null元素吧，只提供了前后trim的方法（如果我想对中间元素进行trim呢）?

![img](http://www.ao10001.wang/group1/M00/00/00/rB9p_Fxz9T2APAdxAAA1KHhm3Z4042.png)

> **Joiner是连接器，Splitter是分割器**，通常会把它们定义为static final，利用on生成对象后在应用到String进行处理，这是可以复用的。要知道apache commons StringUtils提供的都是static method。更加重要的是，guava提供的Joiner/Splitter是经过充分测试，它的稳定性和效率要比apache高出不少。
>
> 对于Joiner，常用的方法是：跳过NULL元素：`skipNulls()` / 对于NULL元素使用其他替代：`useForNull(String)`
>
> 对于Splitter，常用的方法是： `trimResults()`/`omitEmptyStrings()`。注意拆分的方式，有字符串，还有正则，还有固定长度分割.

其实除了Joiner/Splitter外，guava还提供了字符串匹配器：`CharMatcher`

![img](http://www.ao10001.wang/group1/M00/00/00/rB9p_Fxz9oKAaumdAAA5-PZbZtM219.png)

> **CharMatcher**，将字符的匹配和处理解耦。

```
String noControl = CharMatcher.javaIsoControl.removeFrom(string); //移除control字符
String theDigits = CharMatcher.DIGIT.retainFrom(string); //只保留数字字符
String spaced = CharMatcher.WHITESPACE.trimAndCollapseFrom(string, ' ');
//去除两端的空格，并把中间的连续空格替换成单个空格
String noDigits = CharMatcher.JAVA_DIGIT.replaceFrom(string, "*"); //用*号替换所有数字
String lowerAndDigit = CharMatcher.JAVA_DIGIT.or(CharMatcher.JAVA_LOWER_CASE).retainFrom(string);
// 只保留数字和小写字母
```

#### 二、对基本类型进行支持

guava对JDK提供as型操作进行了扩展，使得功能更加强大！

![img](http://www.ao10001.wang/group1/M00/00/00/rB9p_Fxz_ACAbSSZAAA30tx84G4361.png)

> guava提供了Bytes/Shorts/Ints/Iongs/Floats/Doubles/Chars/Booleans这些基本数据类型的扩展支持，只有你想不到的，没有它没有的！

#### 三、对JDK集合的有效补充

灰色地带:`Multiset`

> JDK的集合，提供了**有序且可以重复的List**，**无序且不可以重复的Set**。那这里其实对于集合涉及到了2个概念，一个order，一个dups。那么List vs Set，and then some 。
>
> Multiset就是`无序的`，但是`可重复`的集合，它就是游离在List/Set之间的“灰色地带”！（至于有序的，不允许重复的集合嘛，guava还没有提供）

```
public void demo01() {
    Multiset multiset = HashMultiset.create();
    multiset.add("a");
    multiset.add("a");
    multiset.add("b");
    multiset.add("c");
    multiset.add("b");
    System.out.println(multiset.size()); //5
    System.out.println(multiset.count("a")); //2
}
```

> Multiset自带一个有用的功能，就是可以跟踪每个对象的数量。

#### 四、Immutable vs unmodifiable

```
public void demo02() {
    //list的不可变性设置
    List<String> list = new ArrayList<>();
    list.add("a");
    list.add("b");

    List<String> readOnlyList = Collections.unmodifiableList(list);
    //readOnlyList.add("c"); //UnsupportedOperationException
    list.add("c");
    System.out.println(readOnlyList.size()); //3
}
```

> 实际上，**Collections.unmodifiableXxx**所返回的集合和源集合是同一个对象，只不过可以对集合做出改变的API都被override，会抛出UnsupportedOperationException。
>
> 也即是说我们改变源集合，导致不可变视图（unmodifiable View）也会发生变化.

当然，在不使用guava的情况下，我们是怎么避免上面的问题的呢？

```
public void demo03(){
    //list的不可变性设置
    List<String> list = new ArrayList<>();
    list.add("a");
    list.add("b");

    List<String> readOnlyList = Collections.unmodifiableList(new ArrayList<>(list));
    list.add("c");
    System.out.println(readOnlyList.size()); //2
}
```

> Defensive Copies，保护性拷贝。

那么Guava是如何做的呢？

```
public void demo04() {
    //list的不可变性设置
    List<String> immutableList = ImmutableList.of("a", "b", "c");
    //immutableList.add("111"); //UnsupportedOperationException

    List<String> list = new ArrayList<>();
    list.add("a");
    list.add("b");

    List<String> immutableList2 = ImmutableList.copyOf(list);
    list.add("c");
    //视图不随着源而改变
    System.out.println(list.size() + "---" + immutableList2.size()); //3---2
}
```

#### 五、可不可以一对多：Multimap

> JDK提供给我们的Map是一个键，一个值，一对一的，那么在实际开发中，显然存在一个KEY多个VALUE的情况（比如一个分类下的书本），往往这样表达：Map<k,List>，好像有点臃肿！臃肿也就算了，更加不爽的事，还得判断KEY是否存在来决定是否new 一个LIST出来，有点麻烦！更加麻烦的事情还在后头，比如遍历，比如删除.

来看guava如何替你解决这个大麻烦的：

```
public void demo05() {
    Multimap<String, String> multimap = ArrayListMultimap.create();
    multimap.put("wangao", "man");
    multimap.put("wangao", "yes");
    multimap.put("ao10001", "woman");
    System.out.println(multimap.get("wangao")); //[man, yes]
}
```

> guava所有的集合都有**create**方法，这样的好处在于简单，而且我们不必在重复泛型信息了。
>
> get()/keys()/keySet()/values()/entries()/asMap()都是非常有用的返回view collection的方法。
>
> Multimap的实现类有：ArrayListMultimap/HashMultimap/LinkedHashMultimap/TreeMultimap/ImmutableMultimap等等

#### 六、可不可以双向：BiMap

> JDK提供的MAP让我们可以find value by key，那么能不能通过find key by value呢，能不能KEY和VALUE都是唯一的呢。这是一个双向的概念，即forward+backward。

```
public void demo06() {
    BiMap<String,String> biMap = HashBiMap.create();
    biMap.put("name","wangao");
    //biMap.put("nickName","wangao"); //java.lang.IllegalArgumentException: value already present: wangao
    //强制覆盖
    biMap.forcePut("nickName","wangao");
    biMap.put("123","1111@qq.com");
    System.out.println(biMap.inverse().get("1111@qq.com")); //123
    System.out.println(biMap.inverse().inverse().get("1111@qq.com")); //null
}
```

> biMap / biMap.inverse() / biMap.inverse().inverse() 它们是什么关系呢？
>
> biMap.inverse() != biMap ； biMap.inverse().inverse() == biMap

#### 七、函数式编程：Functions

![img](http://www.ao10001.wang/group1/M00/00/00/rB9p_Fxz_sCAHZARAABDGEdqftM738.png)

> 上面的代码是为了完成将List集合中的元素，先截取5个长度，然后转成大写。
>
> 函数式编程的好处在于在集合遍历操作中提供自定义Function的操作，比如transform转换。其实这个我感觉还是JDK1.8好一点呢。链式操作。

以上就是Guava的冰山一角，以后经常用用，挺方便的。很强的样子。。。。

