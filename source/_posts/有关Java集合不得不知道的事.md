---
layout: posts
title: 有关Java集合不得不知道的事
date: 2018-08-22 00:38:36
categories: 集合
tags: [集合]
top: true
---
说起集合，我想没人没用过，项目中使用集合的概率基本上就是百分百，那么其实对于集合，其实常用的也就那几个，特别是HashMap这个集合，说起Map集合，肯定第一个就是想到的这个集合，那么有些事就不得不知道了，只有真正去了解了，才能更好的使用它，你说对吧？就这篇文章我就来写写集合的那些事。

<!--more--> 

![img](http://www.ao10001.wang/group1/M00/00/01/rB9p_Fx6ODGAOzVKAADran7DYPs838.png)

#### 一、List 和 Set 区别

在Java中除了Map以外的集合的根接口都是`Collection`接口，而在Collection接口的子接口中，最重要的莫过于`List`和`Set`集合接口。

1. List中提供索引的方式来添加元素和获取元素，而Set并不提供。List集合可以精确的存储和获取；Set只能一个一个的比较，显然效率和实用性是比不上List集合的。
2. List集合是有序存储，Set集合是无序存储。这里的有序和无序针对的是存储地址来说的。
3. List可以存储重复的值，Set不可以存储重复的值。

> 说明一下：
>
> Set集合里面遍历出来的值是有序的（从小到大），而List结合遍历出来的值是无序的。
> 但是怎么说Set结合是无序集合而List结合是有序集合呢。 因为我们说的有序和无序针对的是集合存储地址来说的。
> 在一百次循环中，获取什么值，就往List所获取的地址中存储，从低位到到位，比如：第一个是99，99就存储在第一位，第二位是2，就存储在第二位…以此类推
> 而对于Set集合来说，由于是通过HashSet的初始化它的，所以它的存储方式与HashSet存储方式有关，都是根据Hash码来存储的。例如：获取的值是88，那它就把这个值存储在Hash码为88的地址，如果是9，就把这个值存储在Hash码为9的地址，而遍历的时候，Set是根据Hash码的大小来遍历的，所以呈现出来的就是有序的效果。但是在底层存储的时候它是乱序的。

#### 二、Set集合的hashCode以及equals方法

 Set集合中存放的数据有一个特点，那就是无序且不重复。无序是因为Set集合是根据Hash码来存储的中的元素的，是没有下坐标的。不重复的原因就是Set集合中有`hashcode`与`equals`这两个方法。

 `hashCode`方法实际上返回的就是`对象存储的物理地址`。这样一来，当集合要添加新的元素时，先调用这个元素的hashCode方法，就一下子能定位到它应该放置的物理位置上。如果这个位置上没有元素，它就可以直接存储在这个位置上，不用再进行任何比较了；如果这个位置上已经有元素了，就调用它的equals方法与新元素进行比较，相同的话就不存了，不相同就散列其它的地址。

> hashCode不同时，则必为不同对象。hashCode相同时，根据equlas()方法判断是否为同一对象。

#### 三、List 和 Map 区别

1. List是存储单列数据的集合，Map是存储键和值这样的双列数据的集合。
2. List中存储的数据是有顺序，并且允许重复。Map中存储的数据是无序，键不能重复，值是可以重复。
3. List继承Collection接口，Map不继承Collection接口。

#### 四、Arraylist 与 LinkedList 区别

ArrayList的内部实现是基于内部数组Object[]，所以从概念上讲，它更象数组；但LinkedList的内部实现是基于一组连接的记录，它更象一个链表结构，所以，它们在性能上有很大的差别：

1. 对ArrayList和LinkedList而言，在列表末尾增加一个元素所花的开销都是固定的。对ArrayList而言，主要是在内部数组中增加一项，指向所添加的元素，偶尔可能会导致对数组重新进行分配；而对LinkedList而言，这个开销是统一的，分配一个内部Entry对象。

2. 在ArrayList的中间插入或删除一个元素意味着这个列表中剩余的元素都会被移动；而在LinkedList的中间插入或删除一个元素的开销是固定的。

3. LinkedList不支持高效的随机元素访问。

4. ArrayList的空间浪费主要体现在在list列表的结尾预留一定的容量空间，而LinkedList的空间花费则体现在它的每一个元素都需要消耗相当的空间

5. ArrayList 初始化大小是`10`，扩容规则是(1.6)：`扩容后的大小= 原始大小+原始大小/2 + 1`。

   （例如：原始大小是 10 ，扩容后的大小就是 10 + 5+1 = 16）

   JDK1.7扩容规则是，`扩容后的大小= 原始大小+原始大小/2` 。

   （例如：原始大小是 10 ，扩容后的大小就是 10 + 5 = 15， 15+15/2 = 22）

6. LinkedList是一个双向链表，没有初始化大小，也没有扩容的机制，就是一直在前面或者后面新增就好。

> 当操作是在一列数据的后面添加数据而不是在前面或中间，并且需要随机地访问其中的元素时，使用ArrayList会提供比较好的性能。
>
> 当你的操作是在一列数据的前面或中间添加或删除数据，并且按照顺序访问其中的元素时，就应该使用LinkedList了。

#### 五、ArrayList 与 Vector 区别

1. Vector的方法都是同步的(Synchronized)，是线程安全的(thread-safe)，而ArrayList的方法不是，由于线程的同步必然要影响性能，因此,ArrayList的性能比Vector好。
2. 当Vector或ArrayList中的元素超过它的初始大小时，Vector会将它的容量翻倍，而ArrayList只增加50%的大小，这样ArrayList就有利于节约内存空间。

#### 六、HashMap 和 Hashtable 的区别

`HashMap`和`Hashtable`都实现了`Map`接口，他们主要的区别有：`线程安全性`，`同步(synchronization)`，以及`速度`。

1. HashMap几乎可以等价于Hashtable，除了HashMap是非synchronized的，并可以接受`null`(HashMap可以接受为null的键值(key)和值(value)，而Hashtable则不行)。
2. HashMap是`非synchronized`，而Hashtable是`synchronized`，这意味着Hashtable是`线程安全`的，多个线程可以共享一个Hashtable；而如果没有正确的同步的话，多个线程是不能共享HashMap的。Java 5提供了`ConcurrentHashMap`，它是HashTable的替代，比HashTable的扩展性更好。
3. 由于Hashtable是线程安全的，所以在单线程环境下它比HashMap要慢。如果不需要同步，只需要单一线程，那么使用HashMap性能要好过Hashtable。
4. HashMap不能保证随着时间的推移Map中的元素次序是不变的

> `sychronized`意味着在一次仅有一个线程能够更改Hashtable。就是说任何线程要更新Hashtable时要首先获得同步锁，其它线程要等到同步锁被释放之后才能再次获得同步锁更新Hashtable。

能否让HashMap同步？当然可以

```
Map m = Collections.synchronizeMap(hashMap);
```

#### 七、HashSet 和 HashMap 区别

　　`HashSet`实现了Set接口，它不允许集合中出现重复元素。当提到HashSet时，第一件事就是在将对象存储在HashSet之前，要确保重写`hashCode()`方法和`equals()`方法，这样才能比较对象的值是否相等，确保集合中没有储存相同的对象。如果不重写上述两个方法，那么将使用下面方法默认实现：

```
//方法用在Set添加元素时，如果元素值重复时返回 "false"，如果添加成功则返回"true"
public boolean add(Object obj);
```

　　`HashMap`实现了Map接口，Map接口对键值对进行映射。Map中不允许出现重复的键（Key）。Map接口有两个基本的实现`TreeMap`和`HashMap`。TreeMap保存了对象的`排列次序`，而HashMap不能。HashMap是非线程安全的。

```
//方法用来将元素添加到map中
public Object put(Object Key,Object value)
```

#### 八、HashMap 和 ConcurrentHashMap 的区别

HashMap不是线程安全的，在JDK1.5中，提供了concurrent包，从此Map也有安全的了。ConcurrentHashMap具体是怎么实现线程安全的呢，肯定不可能是每个方法加synchronized，那样就变成了HashTable。

Hashmap本质是数组加链表。根据key取得hash值，然后计算出数组下标，如果多个key对应到同一个下标，就用链表串起来，新插入的在前面。

ConcurrentHashMap在hashMap的基础上，将数据分为多个segment(段)，默认16个，然后每次操作对一个segment(段)加锁，避免多线程锁的几率，提高并发效率。

`ConcurrentHashMap`引入了一个`分段锁`的概念，具体可以理解为把一个大的Map拆分成N个小的HashTable，根据`key.hashCode()`来决定把key放到哪个HashTable中。在ConcurrentHashMap中，就是把Map分成了N个Segment，put和get的时候，都是现根据key.hashCode()算出放到哪个Segment中：

![img](http://www.ao10001.wang/group1/M00/00/01/rB9p_Fx6UmSAXIrjAAA1FwnoJVQ582.png)

![img](http://www.ao10001.wang/group1/M00/00/01/rB9p_Fx6UqOAXRLRAAAa0LKyW70549.png)

#### 九、HashMap 的工作原理及代码实现，什么时候用到红黑树

在JDK1.6，JDK1.7中，HashMap采用`位桶+链表`实现，即使用链表处理冲突，同一hash值的链表都存储在一个链表里。但是当位于一个桶中的元素较多，即hash值相等的元素较多时，通过key值依次查找的效率较低。而JDK1.8中，HashMap采用`位桶+链表+红黑树`实现，当链表长度超过`阈值（8）`时，将链表转换为`红黑树`，这样大大减少了查找时间。

##### 1、HashMap的实现原理

首先有一个每个元素都是链表（可能表述不准确）的数组，当添加一个元素（key-value）时，就首先计算元素key的hash值，以此确定插入数组中的位置，但是可能存在同一hash值的元素已经被放在数组同一位置了，这时就添加到同一hash值的元素的后面，他们在数组的同一位置，但是形成了链表，同一个链表上的Hash值是相同的，所以说数组存放的是链表。而当链表长度太长时，链表就转换为红黑树。

当链表数组的容量超过初始容量的0.75倍时，再散列将链表数组扩大2倍，把原链表数组的搬移到新的数组中。

##### 2、JDK1.8中的涉及到的数据结构

1. 位桶数组

   ```java
   //存储（位桶）的数组
   transient Node[] table;
   ```

2. 数组元素Node实现了Entry接口

   ```java
   //Node是单向链表，它实现了Map.Entry接口
   static class Node<K,V> implements Map.Entry<K,V> {
           final int hash;
           final K key;
           V value;
           Node<K,V> next;
   		//构造函数Hash值 键 值 下一个节点
           Node(int hash, K key, V value, Node<K,V> next) {
               this.hash = hash;
               this.key = key;
               this.value = value;
               this.next = next;
           }
   
           public final K getKey()        { return key; }
           public final V getValue()      { return value; }
           public final String toString() { return key + "=" + value; }
   
           public final int hashCode() {
               return Objects.hashCode(key) ^ Objects.hashCode(value);
           }
   
           public final V setValue(V newValue) {
               V oldValue = value;
               value = newValue;
               return oldValue;
           }
   		//判断两个node是否相等,若key和value都相等，返回true。可以与自身比较为true
           public final boolean equals(Object o) {
               if (o == this)
                   return true;
               if (o instanceof Map.Entry) {
                   Map.Entry<?,?> e = (Map.Entry<?,?>)o;
                   if (Objects.equals(key, e.getKey()) &&
                       Objects.equals(value, e.getValue()))
                       return true;
               }
               return false;
           }
       }
   ```

   1. 红黑树

      ```java
      //红黑树
      static final class TreeNode extends LinkedHashMap.Entry {
          TreeNode parent;  // 父节点
          TreeNode left; 	  //左子树
          TreeNode right;	  //右子树
          TreeNode prev;    //needed to unlink next upon deletion
          boolean red;      //颜色属性
          TreeNode(int hash, K key, V val, Node next) {
              super(hash, key, val, next);
          }
          //返回当前节点的根节点
          final TreeNode root() {
              for (TreeNode r = this, p;;) {
                  if ((p = r.parent) == null)
                      return r;
                  r = p;
              }
          }
          ....
      }
      ```

##### 3、源码中的数据域

加载因子（默认0.75）：为什么需要使用加载因子，为什么需要扩容呢？因为如果填充比很大，说明利用的空间很多，如果一直不进行扩容的话，链表就会越来越长，这样查找的效率很低，因为链表的长度很大（当然最新版本使用了红黑树后会改进很多），扩容之后，将原来链表数组的每一个链表分成奇偶两个子链表分别挂在新链表数组的散列位置，这样就减少了每个链表的长度，增加查找效率。

HashMap本来是`以空间换时间`，所以填充比没必要太大。但是填充比太小又会导致空间浪费。如果关注内存，填充比可以稍大，如果主要关注查找性能，填充比可以稍小。

```java
public class HashMap extends AbstractMap implements Map, Cloneable, Serializable {
    private static final long serialVersionUID = 362498820763181265L;
    static final int DEFAULT_INITIAL_CAPACITY = 16;  // aka 16
    static final int MAXIMUM_CAPACITY = 1073741824;  //最大容量
    static final float DEFAULT_LOAD_FACTOR = 0.75f;  //填充比
    //当add一个元素到某个位桶，其链表长度达到8时将链表转换为红黑树
    static final int TREEIFY_THRESHOLD = 8;
    static final int UNTREEIFY_THRESHOLD = 6;
    static final int MIN_TREEIFY_CAPACITY = 64;
    transient Node[] table;//存储元素的数组
    transient Set> entrySet;
    transient int size;//存放元素的个数
    transient int modCount;//被修改的次数fast-fail机制
    int threshold;//临界值 当实际大小(容量*填充比)超过临界值时，会进行扩容 
    final float loadFactor;//填充比（......后面略）
```

##### 4、HashMap的构造函数

HashMap的构造方法有4种，主要涉及到的参数有，`指定初始容量`，`指定填充比`和用来`初始化的Map`；

```java
//构造函数1
public HashMap(int initialCapacity, float loadFactor) {
    //指定的初始容量非负
    if (initialCapacity < 0)
        throw new IllegalArgumentException(Illegal initial capacity:  +
                                           initialCapacity);
    //如果指定的初始容量大于最大容量,置为最大容量
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    //填充比为正
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException(Illegal load factor:  +
                                           loadFactor);
    this.loadFactor = loadFactor;
    this.threshold = tableSizeFor(initialCapacity);//新的扩容临界值
}

//构造函数2
public HashMap(int initialCapacity) {
    this(initialCapacity, DEFAULT_LOAD_FACTOR);
}

//构造函数3
public HashMap() {
    this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
}

//构造函数4用m的元素初始化散列映射
public HashMap(Map m) {
    this.loadFactor = DEFAULT_LOAD_FACTOR;
    putMapEntries(m, false);
}
```

##### 5、HashMap的存取机制

```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}
final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict) {
        Node<K,V>[] tab; 
    	Node<K,V> p; 
    	int n, i;
        //如果tab还未初始化或者table的长度为0先调用resize初始化table
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        //如果table的在（n-1）& hash的值是空，就新建一个节点插入在该位置
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        //否则存在冲突，开始处理冲突
        else {
            Node<K,V> e; K k;
            //检查第一个Node，p是不是要找的值
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
            	//遍历到列表的最后一个，指针为空就挂在后面
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
						//如果冲突的节点数已经达到TREEIFY_THRESHOLD(8)个，
                        //看是否需要改变冲突节点的存储结构　　　　　　　　
                        //treeifyBin首先判断当前hashMap的长度，如果不足64，只进行
                        //resize，扩容table，如果达到64，那么将冲突的存储结构为红黑树
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    //如果有相同的key值就结束遍历
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            //就是链表上有相同的key值
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                //onlyIfAbsent为true那么久不执行重复key的value的替换，
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                //返回存在的Value值
                return oldValue;
            }
        }
        ++modCount;
        //如果当前大小大于门限，门限原本是初始容量*0.75
        if (++size > threshold)
        //扩容两倍
            resize();
        afterNodeInsertion(evict);
        return null;
    }

final void treeifyBin(Node<K,V>[] tab, int hash) {
    int n, index; Node<K,V> e;
    if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
        //如果table的长度不超过MIN_TREEIFY_CAPACITY（=64）个，那么直接进行resize
        resize();
    else if ((e = tab[index = (n - 1) & hash]) != null) {
        TreeNode<K,V> hd = null, tl = null;
        do {
            TreeNode<K,V> p = replacementTreeNode(e, null);
            if (tl == null)
                hd = p;
            else {
                p.prev = tl;
                tl.next = p;
            }
            tl = p;
        } while ((e = e.next) != null);
        if ((tab[index] = hd) != null)
            hd.treeify(tab);
    }
}
```

下面简单说下添加键值对put(key,value)的过程：

1. 判断键值对数组tab[]是否为空或为null，否则以默认大小resize()。
2. 根据键值key计算hash值得到插入的数组索引i，如果tab[i]==null，直接新建节点添加，否则转入3。
3. 判断当前数组中处理hash冲突的方式为链表还是红黑树(check第一个节点类型即可)，分别处理。

##### 6、HasMap的扩容机制resize()

构造hash表时，如果不指明初始大小，默认大小为16（即Node数组大小16），如果Node[]数组中的元素达到（填充比*Node.length）重新调整HashMap大小变为原来2倍大小，扩容很耗时。

```java
//HasMap的扩容机制resize()
final Node[] resize() {
    Node[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    //如果旧表的长度不是空
    if (oldCap > 0) {
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        //把新表的长度设置为旧表长度的两倍，newCap=2*oldCap
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            //把新表的门限设置为旧表门限的两倍，newThr=oldThr*2
            newThr = oldThr << 1; // double threshold
    }
    //如果旧表的长度的是0，就是说第一次初始化表
    else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;
    else {               // zero initial threshold signifies using defaults
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor; //新表长度乘以加载因子
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
    //下面开始构造新表，初始化表中的数据
    Node[] newTab = (Node[])new Node[newCap];
    //把新表赋值给table
    table = newTab; 
    //原表不是空要把原表中数据移动到新表中   
    if (oldTab != null) {
        //遍历原来的旧表  
        for (int j = 0; j < oldCap; ++j) {
            Node e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                //说明这个node没有链表直接放在新表的e.hash & (newCap - 1)位置
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                else if (e instanceof TreeNode)
                    ((TreeNode)e).split(this, newTab, j, oldCap);
                //如果e后边有链表,到这里表示e后面带着个单链表，需要遍历单链表，将每个结点重
                else { // preserve order保证顺序
                    //新计算在新表的位置，并进行搬运
                    Node loHead = null, loTail = null;
                    Node hiHead = null, hiTail = null;
                    Node next;
                    do {
                        next = e.next;//记录下一个结点
                        //新表是旧表的两倍容量，实例上就把单链表拆分为两队，
                        //e.hash&oldCap为偶数一队，e.hash&oldCap为奇数一对
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);

                    if (loTail != null) {//lo队不为null，放在新表原位置
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    if (hiTail != null) {//hi队不为null，放在新表j+oldCap位置
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}
```

##### 7、JDK1.8使用红黑树的改进

在JDK1.8中对HashMap的源码进行了优化，在JDK1.7中，HashMap处理“碰撞”的时候，都是采用链表来存储，当碰撞的结点很多时，查询时间是O（n）。

在JDK1.8中，HashMap处理“碰撞”增加了红黑树这种数据结构，当碰撞结点较少时，采用链表存储，当较大时（阈值为8），采用红黑树（特点是查询时间是O（logn））存储（有一个阀值控制，大于阀值(8个)，将链表存储转换成红黑树存储）。

![img](http://www.ao10001.wang/group1/M00/00/01/rB9p_Fx6Xf2AMEtOAAAT_4MOpuY834.png)

##### 8、问题分析

哈希碰撞会对HashMap的性能带来灾难性的影响。如果多个hashCode()的值落到同一个桶内的时候，这些值是存储到一个链表中的。最坏的情况下，所有的key都映射到同一个桶中，这样Hashmap就退化成了一个链表——查找时间从O(1)到O(n)。

随着HashMap的大小的增长，get()方法的开销也越来越大。由于所有的记录都在同一个桶里的超长链表内，平均查询一条记录就需要遍历一半的列表。

JDK1.8HashMap的红黑树是这样解决的：

如果某个桶中的记录过大的话（当前是TREEIFY_THRESHOLD = 8），HashMap会动态的使用一个专门的Treemap实现来替换掉它。这样做的结果会更好，是O(logn)，而不是糟糕的O(n)。

它是如何工作的？前面产生冲突的那些KEY对应的记录只是简单的追加到一个链表后面，这些记录只能通过遍历来进行查找。但是超过这个阈值后HashMap开始将列表升级成一个二叉树，使用哈希值作为树的分支变量，如果两个哈希值不等，但指向同一个桶的话，较大的那个会插入到右子树里。如果哈希值相等，HashMap希望key值最好是实现了Comparable接口的，这样它可以按照顺序来进行插入。这对HashMap的key来说并不是必须的，不过如果实现了当然最好。如果没有实现这个接口，在出现严重的哈希碰撞的时候，你就并别指望能获得性能提升了。

> HashMap的底层通过位桶实现，位桶里面存的是链表（1.7以前）或者红黑树（有序，1.8开始） ，其实就是数组加链表（或者红黑树）的格式，通过判断hashCode定位位桶中的下标，通过equals定位目标值在链表中的位置，所以如果你使用的key使用可变类（非final修饰的类），那么你在自定义hashCode和equals的时候一定要注意要满足：如果两个对象equals那么一定要hashCode相同，如果是hashCode相同的话不一定要求equals！所以一般来说不要自定义hashCode和equls，推荐使用不可变类对象做key，比如Integer、String等等。