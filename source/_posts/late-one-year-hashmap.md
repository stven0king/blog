---
title: 迟到一年HashMap解读
date: 2017-10-27 11:35:00
tags: [HashMap,hash]
categories: 
description: "HashMap和List这两个类是我们在Java语言编程时使用的频率非常高集合类。“知其然，更要知其所以然”。"
---
前言
===
HashMap和List这两个类是我们在Java语言编程时使用的频率非常高集合类。“知其然，更要知其所以然”。HashMap认识我已经好多年了，对我在工作中一直也尽心尽力的提供帮助。我从去年开始就想去它家拜访来着，可是经常因为各种各样的原因让其遗忘在路过的风景中。（文章大部分源码基于jdk1.7）。

<center>![Map&Set](http://upload-images.jianshu.io/upload_images/1319879-f243ffd6da9f703c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</center>

HashMap概述：
===
 HashMap是基于哈希表实现的键值对的集合，继承自AbstractMap并的Map接口的非同步实现。此实现提供所有可选的映射操作，并允许使用null值和null键。此类不保证映射的顺序，特别是它不保证该顺序恒久不变。
HashMap的特殊存储结构使得在获取指定元素的前需要经过哈希运算，得到目标元素在哈希表中的位置，然后再进行少量的比较即可得到元素，这使得HashMap的查找效率很高。

HashMap的特点
===
- 底层实现JDK1.8之前是数组加链表，之后是数组加红黑树。
- key是用Set进行存储的，所以不允许重复（可以允许null作为key）。
- 元素的存储是无序的，每次重新扩容元素位置可能改变。
- 插入、获取的时间复杂度基本是O(1)（提前试有适当的哈希函数，让元素均匀分布分布）。
- 两个关键因子：出事容量，加载因子。

HashMap的数据结构
===
```java
public class HashMap<K,V>
    extends AbstractMap<K,V>
    implements Map<K,V>, Cloneable, Serializable
{
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16
    static final int MAXIMUM_CAPACITY = 1 << 30;
    static final float DEFAULT_LOAD_FACTOR = 0.75f;
    static final Entry<?,?>[] EMPTY_TABLE = {};
    transient Entry<K,V>[] table = (Entry<K,V>[]) EMPTY_TABLE;
    transient int size;
    int threshold;
    final float loadFactor;
    transient int modCount;
    static final int ALTERNATIVE_HASHING_THRESHOLD_DEFAULT = Integer.MAX_VALUE;
    /**********部分代码省略**********/
    static class Entry<K,V> implements Map.Entry<K,V> {
        final K key;
        V value;
        Entry<K,V> next;
        int hash;
        /**********部分代码省略**********/
    }
    /**********部分代码省略**********/
}
```
HashMap中主要存储着一个Entry的数组table，Entry就是数组中的元素，Entry实现了Map.Entry所以其实Entry就是一个key-value对，并且它持有一个指向下一个元素的引用，这样构成了链表（在java8中Entry改名为Node，因为在Java8中Entry不仅有链表形式还有树型结构，对应的类为TreeNode）。
<center>![HashMap的数据结构](http://upload-images.jianshu.io/upload_images/1319879-7966cdd20c44eff4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</center>

HashMap的构造
===
```java
/**
 * Constructs an empty <tt>HashMap</tt> with the specified initial
 * capacity and load factor.
 *
 * @param  initialCapacity the initial capacity
 * @param  loadFactor      the load factor
 * @throws IllegalArgumentException if the initial capacity is negative
 *         or the load factor is nonpositive
 */
public HashMap(int initialCapacity, float loadFactor) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal initial capacity: " +
                                            initialCapacity);
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal load factor: " +
                                            loadFactor);

    this.loadFactor = loadFactor;
    threshold = initialCapacity;
    init();
}
public HashMap(int initialCapacity) {
    this(initialCapacity, DEFAULT_LOAD_FACTOR);
}
public HashMap() {
    this(DEFAULT_INITIAL_CAPACITY, DEFAULT_LOAD_FACTOR);
}
public HashMap(Map<? extends K, ? extends V> m) {
    this(Math.max((int) (m.size() / DEFAULT_LOAD_FACTOR) + 1,
                    DEFAULT_INITIAL_CAPACITY), DEFAULT_LOAD_FACTOR);
    inflateTable(threshold);

    putAllForCreate(m);
}
```

主要有两个参数，【initialCapacity】初始容量、【loadFactor】加载因子。这两个属性在类定义时候都赋有默认值分别为16和0.75。table数组默认值为EMPTY_TABLE，在添加元素的时候判断table是否为EMPTY_TABLE来调用【inflateTable】。在构造HashMap实例的时候默认【threshold】阈值等于初始容量。当构造方法的参数为Map时，调用 【inflateTable(threshold)】方法对table数组容量进行设置：

```java
/**
 * Inflates the table.
 */
private void inflateTable(int toSize) {
    // Find a power of 2 >= toSize
    int capacity = roundUpToPowerOf2(toSize);
    //更新阈值
    threshold = (int) Math.min(capacity * loadFactor, MAXIMUM_CAPACITY + 1);
    table = new Entry[capacity];
    initHashSeedAsNeeded(capacity);
}
//返回一个比初始容量大的最小的2的幂数,如果number为2的整数幂值那么直接返回，最小为1，最大为2^31。
private static int roundUpToPowerOf2(int number) {
    // assert number >= 0 : "number must be non-negative";
    return number >= MAXIMUM_CAPACITY
            ? MAXIMUM_CAPACITY
            : (number > 1) ? Integer.highestOneBit((number - 1) << 1) : 1;
}
```

highestOneBit
---
返回一个不大于i的2的整数次幂
```java
public static int highestOneBit(int i) {
    // HD, Figure 3-1
    i |= (i >>  1);//i的二进制右边2位为1 。
    i |= (i >>  2);//i的二进制右边4位为1。
    i |= (i >>  4);//i的二进制右边8位为1。
    i |= (i >>  8);//i的二进制右边16位为1。
    i |= (i >> 16);//i的二进制右边32位为1。
    //这样5次移位后再进行与操作，i的所有非0低位全部变成1；
    return i - (i >>> 1);//i减去所有底位的1，只留一个高为的1
}
```
为什么桶的容量要是2的指数，后面会讲到这样有助于添加元素时减少哈希冲突。

HashMap的存取实现
===

HashMap的put方法
---

>- 获取key的hashcode
>- 二次hash
>- 通过hash找到对应的index
>- 插入链表

```java
//HashMap添加元素
public V put(K key, V value) {
    //table没有初始化size=0，先调用inflateTable对table容器进行扩容
    if (table == EMPTY_TABLE) {
        inflateTable(threshold);
    }
    //在hashMap增加key=null的键值对
    if (key == null)
        return putForNullKey(value);
    //计算key的哈希值
    int hash = hash(key);
    //计算在table数据中的bucketIndex
    int i = indexFor(hash, table.length);
    //遍历table[i]的链表，如果节点不为null，通过循环遍历链表的下一个元素
    for (Entry<K,V> e = table[i]; e != null; e = e.next) {
        Object k;
        //找到对应的key，则将value进行替换
        if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
            V oldValue = e.value;
            e.value = value;
            e.recordAccess(this);
            return oldValue;
        }
    }
    //没有找到对应的key的Entry，则需要对数据进行modify,modCount加一
    modCount++;
    //将改key，value添加入table中
    addEntry(hash, key, value, i);
    return null;
}
//添加Entry
void addEntry(int hash, K key, V value, int bucketIndex) {
    //当前桶的长度大于于阈值，而且当前桶的索引位置不为null。则需要对桶进行扩容
    if ((size >= threshold) && (null != table[bucketIndex])) {
        //对桶进行扩容
        resize(2 * table.length);
        //重新计算hash值
        hash = (null != key) ? hash(key) : 0;
        //重新计算当前需要插入的桶的位置
        bucketIndex = indexFor(hash, table.length);
    }
    //在bucketIndex位置创建Entry
    createEntry(hash, key, value, bucketIndex);
}
//创建Entry
void createEntry(int hash, K key, V value, int bucketIndex) {
    //找到当前桶的当前链表的头节点
    Entry<K,V> e = table[bucketIndex];
    //新创建一个Entry将其插入在桶的bucketIndex位置的链表的头部
    table[bucketIndex] = new Entry<>(hash, key, value, e);
    size++;
}
```
获取key的hashcode并进行二次hash
---

```java
final int hash(Object k) {
    int h = hashSeed;
    if (0 != h && k instanceof String) {
        return sun.misc.Hashing.stringHash32((String) k);
    }

    h ^= k.hashCode();

    // This function ensures that hashCodes that differ only by
    // constant multiples at each bit position have a bounded
    // number of collisions (approximately 8 at default load factor).
    h ^= (h >>> 20) ^ (h >>> 12);
    return h ^ (h >>> 7) ^ (h >>> 4);
}
```
为什么这么进行二次hash，目的是唯一的就是让产生的hashcode散列均匀。在网络上也找了一些关于hash值获取的介绍，下边是我找到感觉比较靠谱的一篇文章中关于hash算法的解析：

假设h^key.hashCode()的值为：0x7FFFFFFF，table.length为默认值16。
上面算法执行
<center>![image.png](http://upload-images.jianshu.io/upload_images/1319879-e14ca1e958bf33cf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000)</center>

得到i=15 
其中h\^(h>>>7)^(h>>>4) 结果中的位运行标识是把h>>>7 换成 h>>>8来看。 
即最后h\^(h>>>8)^(h>>>4) 运算后hashCode值每位数值如下： 
> 8=8
> 7=7^8
> 6=6^7^8
> 5=5^8^7^6 
> 4=4^7^6^5^8 
> 3=3^8^6^5^8^4^7 ------------>   3^4^5^6^7 
> 2=2^7^5^4^7^3^8^6 --------->   2^3^4^5^6^8 
> 1=1^6^4^3^8^6^2^7^5  ------>   1^2^3^4^5^7^8 
算法中是采用(h>>>7)而不是(h>>>8)的算法，应该是考虑1、2、3三位出现重复位^运算的情况。使得最低位上原hashCode的8位都参与了\^运算，所以在table.length为默认值16的情况下面，hashCode任意位的变化基本都能反应到最终hash table 定位算法中，这种情况下只有原hashCode第3位高1位变化不会反应到结果中，即：0x7FFFF7FF的i=15。 

从整个二次hash的解析过程来看，通过多次位移和多次与操作获取的hashc。每当key的hashcode有任何变化的时候都能影响到二次hash后的底位的不同，这样在下边根据hash获取在桶上的索引的时候最大减少哈希冲突。

获取hash在桶上的索引
---

> 当我们想找一个hash函数想让均匀分布在桶中时，我们首先想到的就是把hashcode对数组长度取模运算，这样一来，元素的分布相对来说是比较均匀的。但是，“模”运算的消耗还是比较大。而JDK中的实现hash根数组的长度-1做一次“&”操作。

```java
//找到当前的hash在桶的分布节点位置
static int indexFor(int h, int length) {
    // assert Integer.bitCount(length) == 1 : "length must be a non-zero power of 2";
    return h & (length-1);
}
```

这里需要讲一下为什么index=h&(length-1)呢？因为HashMap中的数组长度为2的指数。（lenth-1）的值恰好是数组能容纳的最大容量，且在二进制下每位都是1。所以在经过二次hash之后所获取的code，就能通过一次与操作（取hash值的底位）让其分布在table桶中。

HashMap的get方法
---

> 在理解了put之后，get就很简单了。大致思路如下：
> bucket里的第一个节点，直接命中；
>- 如果有冲突，则通过key.equals(k)去查找对应的entry
>- 若为树，则在树中通过key.equals(k)查找，O(logn)；
>- 若为链表，则在链表中通过key.equals(k)查找，O(n)。

```java
//HashMap的get方法
public V get(Object key) {
    //获取key为null的value
    if (key == null)
        return getForNullKey();
    //获取key对应的Entry实例
    Entry<K,V> entry = getEntry(key);

    return null == entry ? null : entry.getValue();
}
//获取Entry
final Entry<K,V> getEntry(Object key) {
    if (size == 0) {
        return null;
    }
    //计算key的hash值
    int hash = (key == null) ? 0 : hash(key);
    //根据hash调用indexFor方法找到当前key对应的桶的index，遍历该节点对应的链表
    for (Entry<K,V> e = table[indexFor(hash, table.length)];
         e != null;
         e = e.next) {
        Object k;
        //判断当前Entry的hash、key的hash和Entry的key、key是否相等
        if (e.hash == hash &&
            ((k = e.key) == key || (key != null && key.equals(k))))
            return e;
    }
    return null;
}
```


HashMap的扩容
===

> 当HashMap中的元素越来越多的时候，因为数组的长度是固定的所以hash冲突的几率也就越来越高，桶的节点处的链表就越来越长，这个时候查找元素的时间复杂度相应的增加。为了提高查询的效率，就要对HashMap的数组进行扩容（这是一个常用的操作，数组扩容这个操作也会出现在ArrayList中。），而在HashMap数组扩容之后，最消耗性能的地方就出现了：原数组中的数据必须重新计算其在新数组中的位置，并放进去，这就是resize。

> 当HashMap中的元素个数超过阈值时，就会进行数组扩容，【loadFactor】加载因子的默认值为0.75，【threshold】阈值等于桶长乘以loadFactor这是一个折中的取值。也就是说，默认情况下，数组大小为16，那么当HashMap中元素个数超过16*0.75=12的时候，就把数组的大小扩展为 2*16=32，即扩大一倍，然后重新计算每个元素在数组中的位置。

```java
//HashMap扩容
void resize(int newCapacity) {
    //引用备份
    Entry[] oldTable = table;
    //原来桶的长度
    int oldCapacity = oldTable.length;
    //判断是否已经扩容到极限
    if (oldCapacity == MAXIMUM_CAPACITY) {
        threshold = Integer.MAX_VALUE;
        return;
    }
    //根据容器大小创新的建桶
    Entry[] newTable = new Entry[newCapacity];
    //
    transfer(newTable, initHashSeedAsNeeded(newCapacity));
    //重置桶的引用
    table = newTable;
    //重新计算阈值
    threshold = (int)Math.min(newCapacity * loadFactor, MAXIMUM_CAPACITY + 1);
}
//用于初始化hashSeed参数.
//其中hashSeed用于计算key的hash值，它与key的hashCode进行按位异或运算。
//这个hashSeed是一个与实例相关的随机值，主要用于解决hash冲突。
final boolean initHashSeedAsNeeded(int capacity) {
    boolean currentAltHashing = hashSeed != 0;
    boolean useAltHashing = sun.misc.VM.isBooted() &&
            (capacity >= Holder.ALTERNATIVE_HASHING_THRESHOLD);
    boolean switching = currentAltHashing ^ useAltHashing;
    if (switching) {
        hashSeed = useAltHashing
            ? sun.misc.Hashing.randomHashSeed(this)
            : 0;
    }
    return switching;
}
//桶中数据的迁移
void transfer(Entry[] newTable, boolean rehash) {
    //新的痛长
    int newCapacity = newTable.length;
    for (Entry<K,V> e : table) {
        //遍历桶的没一个节点的链表
        while(null != e) {
            Entry<K,V> next = e.next;
            //重新计算哈希值
            if (rehash) {
                e.hash = null == e.key ? 0 : hash(e.key);
            }
            //找到当前Entry在新桶中的位置
            int i = indexFor(e.hash, newCapacity);
            //将Entry添加在当桶中的bucketIndex处的链表的头部
            e.next = newTable[i];
            //将产生的新链表赋值为桶的bucketIndex处
            newTable[i] = e;
            //遍历当前链表的下一个节点
            e = next;
        }
    }
}
```

>- 假设hash算法就是最简单的 key mod table.length（也就是数组的长度）。
>- 最上面的是old hash 表，其中的Hash表的 size = 2, 所以 key = 3, 7, 5，在mod 2以后碰撞发生在 table[1]
>- 接下来的三个步骤是 Hash表 resize 到4，并将所有的 <key,value> 重新resize到新Hash表的过程

<center>![resize](http://upload-images.jianshu.io/upload_images/1319879-4303ce1a45ea22a0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</center>

> 在HashMap进行扩容的时候有一个点大家发现没，所有Entry的hash值是不需要重新计算的。因为hash值与（length - 1）取的总是hash值的二进制右边底位，扩容一次向左多取一位二进制。


有关HashMap的思考
===

>- 什么时候会使用HashMap？他有什么特点？

是基于Map接口的实现，存储键值对时，它可以接收null的键值，是非同步的，HashMap存储着Entry(hash, key, value, next)对象。

>- 你知道HashMap的工作原理吗？

通过hash的方法，通过put和get存储和获取对象。存储对象时，我们将K/V传给put方法时，它调用hashCode计算hash从而得到bucket位置，进一步存储，HashMap会根据当前bucket的占用情况自动调整容量(超过Load Facotr则resize为原来的2倍)。获取对象时，我们将K传给get，它调用hashCode计算hash从而得到bucket位置，并进一步调用equals()方法确定键值对。如果发生碰撞的时候，Hashmap通过链表将产生碰撞冲突的元素组织起来，在Java 8中，如果一个bucket中碰撞冲突的元素超过某个限制(默认是8)，则使用红黑树来替换链表，从而提高速度。

>- 你知道get和put的原理吗？equals()和hashCode()的都有什么作用？

通过对key的hashCode()进行hashing，并计算下标( n-1 & hash)，从而获得buckets的位置。如果产生碰撞，则利用key.equals()方法去链表或树中去查找对应的节点

>- 你知道hash的实现吗？为什么要这样实现？

在通过hashCode()的高位与底位进行异或，主要是从速度、功效、质量来考虑的，这么做可以在bucket的n比较小的时候，也能保证考虑到高低bit都参与到hash的计算中，同时不会有太大的开销。

>- 如果HashMap的大小超过了负载因子(load factor)定义的容量，怎么办？

如果超过了负载因子(默认0.75)，则会重新resize一个原来长度两倍的HashMap，并且重新调用hash方法。

JDK1.8对HashMap的改进
===

代码实现的不同之处
---

```java
//链表切换为红黑树的阈值
static final int TREEIFY_THRESHOLD = 8;
//红黑树切花为链表的阈值
static final int UNTREEIFY_THRESHOLD = 6;
//红黑树上的节点个数满足时对整个桶进行扩容
static final int MIN_TREEIFY_CAPACITY = 64;
//红黑树
static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
    TreeNode<K,V> parent;  // red-black tree links
    TreeNode<K,V> left;
    TreeNode<K,V> right;
    TreeNode<K,V> prev;    // needed to unlink next upon deletion
    boolean red;
    /*************部分代码省略*****************/
}
//获取key的hashCode,并进行二次hash。二次hash只是将hashcode的高16位于第16位进行异或
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
//resize时hash冲突使用的是红黑树
final Node<K,V>[] resize() {
    /*************部分代码省略*****************/
}
```

性能的提升
---

> 哈希碰撞会对hashMap的性能带来灾难性的影响。如果多个hashCode()的值落到同一个桶内的时候，这些值是存储到一个链表中的。最坏的情况下，所有的key都映射到同一个桶中，这样hashmap就退化成了一个链表——查找时间从O(1)到O(n)，而使用红黑树代替链表查找时间会变为O(logn)。

参考文章：
[主题：HashMap hash方法分析](http://www.iteye.com/topic/709945)

文章到这里就全部讲述完啦，若有其他需要交流的可以留言哦~！~！

想阅读作者的更多文章，可以查看我 [个人博客](http://dandanlove.com/) 和公共号：
<center>![振兴书城](http://upload-images.jianshu.io/upload_images/1319879-612c4c66d40ce855.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)<center>