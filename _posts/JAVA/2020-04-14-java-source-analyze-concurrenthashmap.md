---
layout: post
title: Java 容器之 ConcurrentHashMap 源码分析
excerpt: Java容器之ConcurrentHashMap源码分析（JDK 1.7与JDK 1.8对比）
date: 2020-04-14
author: hestyle
authorSite: https://blog.csdn.net/qq_41855420
firstPostShow: hestyle's blog
firstPostURL: https://blog.csdn.net/qq_41855420/article/details/105492111
color: rgb(1,63,148)
cover: /assets/post/java.jpg
tags: Java
---

&emsp;&emsp;在前面两篇博客 [Java容器之HashMap源码分析](https://hestyle.blog.csdn.net/article/details/105425716)、[Java容器之Hashtable源码分析](https://hestyle.blog.csdn.net/article/details/105448273)分别对JDK1.8中的`HashMap`、`Hashtable`的源码进行一些分析，在本篇博客将对`ConcurrentHashMap`容器的源码进行一些分析。

$\color{red}温馨提示：$此篇博客难度稍大，请阅读完`HashMap`、`Hashtable`的源码分析再来阅读~

**申明**：在前两篇博客介绍了两个容器的`增、删、改、查`相关的API，并且给源码加上了中文注释，在本篇博客将只介绍一些关键的API，比如`hash值`的计算，`put`、`get`、`扩容`等操作。并且由于JDK1.8对`ConcurrentHashMap`进行了较大的改进，所以将会把JDK 1.7与JDK 1.8对比分析。

# 一、`ConcurrentHashMap`容器概述
&emsp;&emsp;在前两两篇博客多次提及到`HashMap`只支持单线程的读写，未使用任何锁相关的机制，所以效率较高，而`Hashtable`支持多线程的读写，底层使用了`synchronized`关键字（非静态同步函数，this锁），因此效率相对较低。那么为什么又要引入`ConcurrentHashMap`容器呢？
&emsp;&emsp;`ConcurrentHashMap`类声明：
```java
package java.util.concurrent;

public class ConcurrentHashMap<K, V> extends AbstractMap<K, V>
        implements ConcurrentMap<K, V>, Serializable
```

&emsp;&emsp;`ConcurrentHashMap`容器相比于`HashMap`，前者支持多线程读写操作，相比于`Hashtable`容器，`ConcurrentHashMap`容器分段管理，对每个段分别加锁（`RententLock`锁机制），减少了锁的粒度，从而效率更高些。
&emsp;&emsp;并且在JDK1.8中，对`ConcurrentHashMap`进行了大幅修改，去除了分段管理策略，锁机制修改为`CAS`+`synchronized`，并且同时引入了红黑树解决hash冲突的问题。具体区别请继续阅读：
# 二、JDK1.7中的`ConcurrentHashMap`容器
## 1、一图以蔽之
&emsp;&emsp;JDK1.7中的`ConcurrentHashMap`容器采用分段管理，初始化构造时默认16个段（可进行动态修改），每个段都有一个`RententLock`锁，并且每个`Segment`分别对应一个`HashEntry`（可以理解为一个小的`Hashtable`）。我们需要删除、增加一个`key-value`时，先判断key对应的hash所在的段，然后再在段中找到响应的位置删除或插入。（如果是多个线程同时进行读写操作，有很大可能是读写不同的段，所以效率较`Hashtable`有较大提升）。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200413204459858.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQxODU1NDIw,size_16,color_FFFFFF,t_70)
## 2、`hash`计算方法
&emsp;&emsp;`hash`计算方法的主要目的是将一个`对象`计算出一个hash值（功能类似Object类的hashCode方法），不过此处的计算要求对于任意给定的`对象`需要将`hash`尽可能的均匀分布，从何降低多个对象（主要指key）的`hash值`一样的情况（hash冲突）。
```java
/**
 * 计算hash值
 */
private int hash(Object k) {
	// hash随机种子
    int h = hashSeed;

    if ((0 != h) && (k instanceof String)) {
        return sun.misc.Hashing.stringHash32((String) k);
    }
	// 异或Object类的hashCode值
    h ^= k.hashCode();

    // 下面是 Wang/Jenkins hash算法（可以自行百度一下这个算法），对h再次进行操作
    // 主要的目的是减少hash散列冲突，分布不均匀
    h += (h <<  15) ^ 0xffffcd7d;
    h ^= (h >>> 10);
    h += (h <<   3);
    h ^= (h >>>  6);
    h += (h <<   2) + (h << 14);
    return h ^ (h >>> 16);
}
```
## 3、`get`方法
&emsp;&emsp;`get`方法的作用是通过key获取value，此方法虽说没有使用任何锁机制，但是`ConcurrentHashMap`容器的Segment中的HashEntry属性使用了`volatile`关键字，即如果有线程修改了某个节点，对于其他线程是可见的，从而保证了正在执行`get`方法的线程获取到的都是最新的值。
```java
/**
 * 通过key获取value
 */
public V get(Object key) {
	// 用于指向key对应的hash值所在的段
    Segment<K,V> s;
    // 用于指向段对应的HashEntry
    HashEntry<K,V>[] tab;
    // 调用hash方法计算hash值
    int h = hash(key);
    // 获取段下标
    // segmentShift是段偏移位数，segmentMask是段掩码
    
    // 假设 h = 1111 0101，容器共有8个段（需要3位二进制，segmentMask = “0000 0111”），并且每个段中的HashEntry长度为16（4位二进制）
    // segmentShift段偏移位数 = 4，把h右移segmentShift位得到 h` = “0000 1111” 段号`
    // 但是只有8个段，此时我们只能取 h` = “0000 1111” 的低三位，
    // 所以与“0000 0111”（即segmentMask段掩码）进行&操作，即 h` & segmentMask
    
    // 至于SSHIFT、SBASE可以不用管，是调用UNSAFE类的一些辅助参数，看懂segmentShift、segmentMask就行
    long u = (((h >>> segmentShift) & segmentMask) << SSHIFT) + SBASE;
    // UNSAFE.getObjectVolatile(segments, u)方法是获取segments数组中偏移量为u的引用，赋值给s
    if ((s = (Segment<K,V>)UNSAFE.getObjectVolatile(segments, u)) != null && (tab = s.table) != null) {
    	// (tab.length - 1) & h 获取h的段内偏移量
    	// 在通过UNSAFE.getObjectVolatile方法，获取对应的引用（key所在的hash桶（HashEntry数组的哪个位置））
    	// 遍历这个链表即可（建议与上面画的结构示意图结合看）
        for (HashEntry<K,V> e = (HashEntry<K,V>) UNSAFE.getObjectVolatile(tab, ((long)(((tab.length - 1) & h)) << TSHIFT) + TBASE); e != null; e = e.next) {
            K k;
            if ((k = e.key) == key || (e.hash == h && key.equals(k)))
                return e.value;
        }
    }
    return null;
}
```
## 4、`put`方法
### ①、`ConcurrentHashMap`中的`put`方法
&emsp;&emsp;`ConcurrentHashMap`类中的`put`方法是对外开放的增加（更新）`key-value`的API。
```java
/**
 * 插入key-value
 */
@SuppressWarnings("unchecked")
public V put(K key, V value) {
    Segment<K,V> s;
    if (value == null)
        throw new NullPointerException();
    int hash = hash(key);
    // 计算 hash对应的段偏移量（segmentShift、segmentMask在get方法中详细介绍了，不清楚的往前翻翻吧）
    int j = (hash >>> segmentShift) & segmentMask;
    if ((s = (Segment<K,V>)UNSAFE.getObject(segments, (j << SSHIFT) + SBASE)) == null) //  in ensureSegment
    	// 如果这个段还为null，则需要初始化一个
        s = ensureSegment(j);
    // 调用Segment中的put方法
    return s.put(key, hash, value, false);
}
```
### ②、`Segment`中的`put`方法
&emsp;&emsp;`Segment`类中的`put`方法是`ConcurrentHashMap`容器中的段对内开放的增加（更新）`key-value`的API。
```java
/**
 * hash是调用ConcurrentHashMap中的hash方法计算key的hash值
 * onlyIfAbsent参数的作用，当前是否仅进行插入操作，
 *     = false时，表示如果容器中已经含有key对应的key-value，直接更新value
 */
final V put(K key, int hash, V value, boolean onlyIfAbsent) {
	// 试图获取该段的lock,失败则调用scanAndLockForPut方法继续尝试获取
    HashEntry<K,V> node = tryLock() ? null :scanAndLockForPut(key, hash, value);
    V oldValue;
    try {
        HashEntry<K,V>[] tab = table;
        // 计算段内偏移量
        int index = (tab.length - 1) & hash;
        // 获取段内的HashEntry数组中的下标对应的引用
        HashEntry<K,V> first = entryAt(tab, index);
        // 遍历这个链表，查找key
        for (HashEntry<K,V> e = first;;) {
            if (e != null) {
                K k;
                if ((k = e.key) == key || (e.hash == hash && key.equals(k))) {
                	// key匹配成功
                    oldValue = e.value;
                    // onlyIfAbsent = false，则说明查找到了需要更新操作
                    //              = true，则表示查找到了不更新，没查找到就插入
                    if (!onlyIfAbsent) {
                        e.value = value;
                        ++modCount;
                    }
                    break;
                }
                e = e.next;
            }
            else {
            	// e == null,即查到了这条链表的尾端
                if (node != null)
                    node.setNext(first);
                else
                	// node为空，说明HashEntry[i]为空，新建
                    node = new HashEntry<K,V>(hash, key, value, first);
                // 判断插入这个节点是否超过 最大容量，需要对进行段进行扩容
                int c = count + 1;
                if (c > threshold && tab.length < MAXIMUM_CAPACITY)
                    rehash(node);
                else
                	// 更新tab[index]
                    setEntryAt(tab, index, node);
                // 插入节点后，key-value数自增，修改数自增
                ++modCount;
                count = c;
                // 新插入节点，肯定没有就值，所以置空
                oldValue = null;
                break;
            }
        }
    } finally {
    	// 最后需要手动释放锁（这是RententLock的特点）
        unlock();
    }
    return oldValue;
}
```

## 5、`remove`方法
### ①、`ConcurrentHashMap`中的`remove`方法
&emsp;&emsp;`ConcurrentHashMap`类中的`remove`方法是对外开放的移除`key-value`的API。
```java
/**
 * 根据key移除key-value
 */
public V remove(Object key) {
    int hash = hash(key);
    // segmentForHash方法的作用是根据hash值获取Segments数组中对应的引用
    Segment<K,V> s = segmentForHash(hash);
    // hash对应的段存在，则调用段中的remove方法
    return s == null ? null : s.remove(key, hash, null);
}
```
### ②、`Segment`中的`remove`方法
&emsp;&emsp;`Segment`类中的`remove`方法是`ConcurrentHashMap`容器中的段对内开放的移除`key-value`的API。

```java
/**
 * 根据key-value移除节点，value = null时，不匹配value
 * 移除成功则返回key对应的旧value
 */
final V remove(Object key, int hash, Object value) {
    if (!tryLock())
    	// 视图获取锁，没有获取成功则调用scanAndLock方法继续尝试
        scanAndLock(key, hash);
    V oldValue = null;
    try {
        HashEntry<K,V>[] tab = table;
        // 计算段内偏移量
        int index = (tab.length - 1) & hash;
        HashEntry<K,V> e = entryAt(tab, index);
        HashEntry<K,V> pred = null;
        while (e != null) {
            K k;
            HashEntry<K,V> next = e.next;
            if ((k = e.key) == key || (e.hash == hash && key.equals(k))) {
                V v = e.value;
                // value == null说明移除key-value的时候成功匹配key就进行移除操作
                // 或者说value与e.value值相同，即value也匹配成功，也需要进行移除
                if (value == null || value == v || value.equals(v)) {
                    // pred（e的前一个节点） == null，说明key在链表的头部，设置table[index] = e.next
                    if (pred == null)
                        setEntryAt(tab, index, next);
                    else
                    	// 否则需要将e移出链表，即e.pre = e.next
                        pred.setNext(next);
                    // 修改次数自增，段中的key-value数自减
                    ++modCount;
                    --count;
                    oldValue = v;
                }
                break;
            }
            // pred用于指向e的前一个节点
            pred = e;
            e = next;
        }
    } finally {
    	// 最后释放锁
        unlock();
    }
    return oldValue;
}
```
## 6、`Segment`中的`rehash`方法
&emsp;&emsp;在`ConcurrentHashMap`容器中，段的数量无法改变，但是段中的`HashEntry`数组的容量可以修改，调用`rehash`即可扩容。

$\color{red}注意：$`rehash`方法只在`Segment`中put节点，并且插入节点后key-value的数量会超过容量阈值的情况下，才会调用此方法。
```java
/**
 * node是待插入的新节点，oldTable已满，无法插入node，对oldTable进行扩容
 */
@SuppressWarnings("unchecked")
private void rehash(HashEntry<K,V> node) {
    // 记录扩容前的table信息
    HashEntry<K,V>[] oldTable = table;
    int oldCapacity = oldTable.length;
    // 每次扩容都是 * 2（注意：在ConcurrentHashMap中同样规定了HashEntry的大小必须是2的次幂，比如2，4，8，16...等）
    int newCapacity = oldCapacity << 1;
    // 同时需要更新key-value阈值（新容量 * 负载因子）
    threshold = (int)(newCapacity * loadFactor);
    HashEntry<K,V>[] newTable = (HashEntry<K,V>[]) new HashEntry[newCapacity];
    // 由于newCapacity也是2的次幂，二进制形式为 010000...000、001000...000这种，减一后就是 001111...111、000111...111这种
    int sizeMask = newCapacity - 1;
    // 遍历oldTable
    for (int i = 0; i < oldCapacity ; i++) {
        HashEntry<K,V> e = oldTable[i];
        // 将oldTable[i]链表下的所有key-value根据key对应的hash进行再散列
        if (e != null) {
            HashEntry<K,V> next = e.next;
            // 获取 e.hash底位，与上sizeMask（二进制形式的低位全为1）即可(相当于求余操作 hash % size)
            int idx = e.hash & sizeMask;
            if (next == null)
            	// e没有next，则是单个节点
                newTable[idx] = e;
            else {
            	// 否则我们还需遍历该链表后面的节点
                HashEntry<K,V> lastRun = e;
                int lastIdx = idx;
                // 找到该链表的尾端
                for (HashEntry<K,V> last = next; last != null; last = last.next) {
                    int k = last.hash & sizeMask;
                    if (k != lastIdx) {
                        lastIdx = k;
                        lastRun = last;
                    }
                }
                // 最后一个节点直接复制到newTable[lastIdx]
                newTable[lastIdx] = lastRun;
                // 将其他节点同样再hash到newTable中
                for (HashEntry<K,V> p = e; p != lastRun; p = p.next) {
                    V v = p.value;
                    int h = p.hash;
                    // 获取 p.hash底位，与上sizeMask（二进制形式的低位全为1）即可(相当于求余操作 hash % size)
                    int k = h & sizeMask;
                    // 将e插入到newTable[k]的头部
                    HashEntry<K,V> n = newTable[k];
                    newTable[k] = new HashEntry<K,V>(h, p.key, v, n);
                }
            }
        }
    }
    // 获取 node.hash底位，与上sizeMask（二进制形式的低位全为1）即可(相当于求余操作 hash % size)
    // 这里可能有部分人有疑问，既然你将oldTable都遍历了一遍，为啥还要插入node呢
    // 这不就是将node重复插入了么，其实rehash方法只有在Segment中的put方法中有调用，也就是说每次调用它时，node都是插入新节点，即它不在oldTable中
    int nodeIndex = node.hash & sizeMask;
    // 将node放到newTable[nodeIndex]的头部
    node.setNext(newTable[nodeIndex]);
    newTable[nodeIndex] = node;
    // 更新table
    table = newTable;
}
```

# 三、JDK1.8中的`ConcurrentHashMap`容器
## 1、一图以蔽之
&emsp;&emsp;在JDK1.8中，移除了`ConcurrentHashMap`容器中的分段管理，只剩下一个`table`数组，并且增加了红黑树结构优化链表过长访问降低效率的问题，与JDK1.8中的HashMap底层数据结构一致。
&emsp;&emsp;那么之前一直提到的`ConcurrentHashMap`相比于`Hashtable`分段管理（分段锁，减小锁的粒度）的优势岂不是没有了？JDK1.8也修改之前的`RententLock`锁机制为`CAS`+`synchronized`，并且尽量减少了使用锁的代码（在`Hashtable`是`synchronized`同步函数，锁的粒度大）。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200413214827859.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQxODU1NDIw,size_16,color_FFFFFF,t_70)
&emsp;&emsp;可能前面的介绍还是有些笼统，为什么`ConcurrentHashMap`高效，`CAS`又是什么操作？
&emsp;&emsp;`CAS`，`Compare and Swap`，主要思想是：三个参数，一个当前内存值V、旧的预期值A、即将更新的值B，当且仅当预期值A和内存值V相同时，将内存值修改为B并返回true，否则什么都不做，并返回false。

下面是`ConcurrentHashMap`类中申明的三个关于`CAS`的方法。

```java
/**
 * 获取tab数组的引用
 */
static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
    return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
}
/**
 * 更新tab[i]，c是更新前的预期值（tab[i]可能被其它线程改动了），v是需要更新为的值
 * 调用compareAndSwapObject方法，更新前先对比tab[i] == c，不相等则说明被其它线程改动了，更新失败，否则更新
 */
static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i, Node<K,V> c, Node<K,V> v) {
    return U.compareAndSwapObject(tab, ((long)i << ASHIFT) + ABASE, c, v);
}
/**
 * 设置tab[i] = v
 */
static final <K,V> void setTabAt(Node<K,V>[] tab, int i, Node<K,V> v) {
    U.putObjectVolatile(tab, ((long)i << ASHIFT) + ABASE, v);
}
```

## 2、主要属性
&emsp;&emsp;下面是`ConcurrentHashMap`类中的主要属性，大概浏览一遍就行，用到的时候再返回来看看。
```java
/**
 * table表的最大长度
 */
private static final int MAXIMUM_CAPACITY = 1 << 30;

/**
 * 默认table表的长度为16（必须是2的次幂）
 */
private static final int DEFAULT_CAPACITY = 16;

/**
 * array数组的最大长度，与toArray方法有关
 */
static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;

/**
 * 并发级别，在jdk1.7中有段的概念，需要分段锁，此为向前兼容
 */
private static final int DEFAULT_CONCURRENCY_LEVEL = 16;

/**
 * table表的负载因子
 */
private static final float LOAD_FACTOR = 0.75f;

/**
 * 若某链表的长度超过该值就需要转换成红黑树（当然还需要一个条件，继续往下看）
 */
static final int TREEIFY_THRESHOLD = 8;

/**
 * 若某红黑树中key-value数少于该值，需要重新转换回链表
 */
static final int UNTREEIFY_THRESHOLD = 6;

/**
 * 前面说如果某链表的长度 > TREEIFY_THRESHOLD 就需要转换为红黑树
 * 实际上还有一个附加条件，就是table数组的长度 不小于 MIN_TREEIFY_CAPACITY
 * 该值主要是考虑到table数组短而造成严重的hash冲突问题，即某链表下的节点数过多
 * 此时首要考虑的是扩大table数组的长度，而不是将链表转换为红黑树
 */
static final int MIN_TREEIFY_CAPACITY = 64;

/**
 * 扩容线程每次最少要迁移的hash桶数（至少是table数组的长度）
 */
private static final int MIN_TRANSFER_STRIDE = 16;


/**
 * The maximum number of threads that can help resize.
 * Must fit in 32 - RESIZE_STAMP_BITS bits.
 */
private static final int MAX_RESIZERS = (1 << (32 - RESIZE_STAMP_BITS)) - 1;


/*
 * 一般标记在hash桶中的第一个元素
 */
static final int MOVED     = -1; // 正在移动
static final int TREEBIN   = -2; // 红黑树
static final int RESERVED  = -3; // hash for transient reservations
/**
 * 在spread方法中有用到，主要是将int型最高位置零，放置hash出现负数，不好进行求余操作
 */
static final int HASH_BITS = 0x7fffffff;

/** CPU个数, to place bounds on some sizings */
static final int NCPU = Runtime.getRuntime().availableProcessors();
    
/* ---------------- Fields -------------- */

/**
 * table数组（hash桶数组），ConcurrentHashMap容器存放key-value的核心
 */
transient volatile Node<K,V>[] table;

/**
 * 临时tab数组，用于正在扩容时的暂时存放
 */
private transient volatile Node<K,V>[] nextTable;

/**
 * 这个值比较重要，也比较复杂，有好几种状态。。。
 * 构造的时候（调用构造方法）
 * 		= 0，未设置（初始化会使用DEFAULT_CAPACITY默认大小）
 * 		> 0,表示的table数组需要初始化的长度（必须是2的次幂）
 * 初始化的时候（调用initTable方法），需要使用构造时候设置的值
 * 		= -1 正在初始化
 * 		> 0(初始化完了，= max(构造的时候的值, DEFAULT_CAPACITY) * 负载因子)，表示容器中存放key-value的阈值
 * 其它值
 *    < -1,分为两部分，高15位是新容量的值，低16位(M)表示并行扩容线程数+1，具体在resizeStamp函数介绍。
 */
private transient volatile int sizeCtl;

/**
 * [0, transferIndex)表示table数组中待移动的hash桶（主要用在扩容时）
 */
private transient volatile int transferIndex;
```
## 3、构造、初始化相关的方法
### ①、构造方法
&emsp;&emsp;jdk1.8中的`ConcurrentHashMap`类一共有构造方法，为缩小篇幅，我只取了其中三个重要的，其它两个一个是默认空构造方法，一个是调用了下面第二个构造方法。
$\color{red}注意：$在构造方法中并没有设定table数组什么的，只是设置了`sizeCtl`，也就是说没有真正的初始化。
```java
public ConcurrentHashMap(int initialCapacity) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException();
    // initialCapacity 超过 MAXIMUM_CAPACITY / 2， 则赋值为最大值 MAXIMUM_CAPACITY（前面属性已经介绍过这个值，2^30 , 不记得的往前面翻翻吧）
    // initialCapacity + (initialCapacity >>> 1) + 1,由于有负载因子的存在，所以你想初始化的大小 ÷ 负载因子（默认是0.75）就是容器table数组真正初始化的长度
    // 否则需要调用tableSizeFor(number)方法，计算不小于number的2的次幂，这个方法在HashMap中介绍过，想要打破砂锅问到底的可以去翻前一篇介绍hashMap的博客
    // 最终 cap 是一个2的次幂
    int cap = ((initialCapacity >= (MAXIMUM_CAPACITY >>> 1)) ? MAXIMUM_CAPACITY : tableSizeFor(initialCapacity + (initialCapacity >>> 1) + 1));
    // 赋值给sizeCtl（前面也介绍过这个属性， 不记得的往前面翻翻吧）
    this.sizeCtl = cap;
}
/**
 * @parm initialCapacity 初始化容量
 * @parm loadFactor 初始化负载因子
 * @parm concurrencyLevel 并发量（类似JDK1.7中的段数，分段锁，主要是向前兼容，可以忽略
 */
public ConcurrentHashMap(int initialCapacity, float loadFactor, int concurrencyLevel) {
	// 检查initialCapacity、loadFactor、concurrencyLevel参数合法性
    if (!(loadFactor > 0.0f) || initialCapacity < 0 || concurrencyLevel <= 0)
        throw new IllegalArgumentException();
    // 初始化容量总不能小于并发量吧。。。
    if (initialCapacity < concurrencyLevel)
        initialCapacity = concurrencyLevel;
    // 你希望初始化的大小是initialCapacity，但是负载因子的存在，容器的table数组的长度应该是 (long)initialCapacity / loadFactor，加上1.0主要是向上取整
    long size = (long)(1.0 + (long)initialCapacity / loadFactor);
    // 超过MAXIMUM_CAPACITY（前面介绍过这个属性），则附最大值，否则调用tableSizeFor
    // tableSizeFor(number)方法，计算不小于number的2的次幂，这个方法在HashMap中介绍过，想要打破砂锅问到底的可以去翻前一篇介绍hashMap的博客
    int cap = (size >= (long)MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : tableSizeFor((int)size);
    // 最终 sizeCtl仍然是2的次幂
    this.sizeCtl = cap;
}

/**
 * 此方法是复制构造方法，需要将容器m中的内容复制到新建的容器
 */
public ConcurrentHashMap(Map<? extends K, ? extends V> m) {
    this.sizeCtl = DEFAULT_CAPACITY;
    // 此方法调用了putVal方法，进而调用initTable初始化方法
    putAll(m);
}
```

### ②、`initTable`初始化方法
&emsp;&emsp;在JDK1.8的实现中，构造方法并没有初始化`table`数组，只有当往容器中放入元素时（调用`put`、`putVal`方法），此时才会进行初始化容器。这里先介绍这个方法，后面有介绍`put`、`putVal`方法，就会看到这个初始化方法在什么时候进行过调用。
```java
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    // while循环检查table是否初始化了
    while ((tab = table) == null || tab.length == 0) {
        if ((sc = sizeCtl) < 0)
        	// 前面介绍sizeCtl属性时，sizeCtl < 0 可能是在扩容或者初始化
        	// 次数tab为null显然不是扩容，只能说明其它线程正在初始化，此时当前线程调用yield方法让出此时运行的CPU时间片
            Thread.yield(); // lost initialization race; just spin
        // 调用Unsafe的compareAndSwapInt方法，更新sizeCtl = -1
        // 不过compareAndSwapInt方法更新前需要判断sizeCtl是否发生了修改，我们之前用sc保存了sizeCtl的值
        // SIZECTL是sizeCtl内存地址（在类中有赋值，不用纠结在哪赋值的，再深入我博客篇幅就太长了。。。）
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
            try {
            	// tab为空，新建nt数组，长度为sc（sizeCtl初始化的值，table数组的长度）
                if ((tab = table) == null || tab.length == 0) {
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    // n >>> 2是n的四分之一，相当于n * 0.75，0.75不正是默认的负载因子么。。。得到的结果是容器可放置的key-value阈值
                    sc = n - (n >>> 2);
                }
            } finally {
            	// 最后sizeCtl更新为容器可放置的key-value阈值
                sizeCtl = sc;
            }
            break;
        }
    }
    return tab;
}
```
## 4、`put`相关的方法
### ①、`put`方法
&emsp;&emsp;`put`方法是对外展示的插入/更新`key-value`的接口。
```java
public V put(K key, V value) {
	// 调用putVal方法，第三个参数表示是否只做单纯的插入，false代表容器中如果存在key对应的value进行更新操作，否则插入
    return putVal(key, value, false);
}
```
### ②、`putVal`方法
&emsp;&emsp;`putVal`方法是对内提供的插入/更新key-value接口，此方法加了锁。
```java
/**
 * @parm onlyIfAbsent 表示是否只做单纯的插入，false代表容器中如果存在key对应的value进行更新操作，否则插入
 */
final V putVal(K key, V value, boolean onlyIfAbsent) {
	// ConcurrentHashMap不支持key/value为null
    if (key == null || value == null) 
    	throw new NullPointerException();
    // 调用spread得到hash值（此方法在下文有介绍）
    int hash = spread(key.hashCode());
    // 用于计算key所在的链表（hash桶）中的key-value数（最后判断是否要改红黑树）
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        // 前面提及过，在插入前需要判断table是否初始化了
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();
        // (n - 1) & hash其实是对hash % n,(多次提到table数组长是2的次幂，就是为了求余方便)
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
        	// table[i] == null,则调用封装好的cas操作，替换table[i]，即插入到这个位置
            if (casTabAt(tab, i, null, new Node<K,V>(hash, key, value, null)))
                break;                   // no lock when adding to empty bin
        }
        else if ((fh = f.hash) == MOVED)
        	// 如果tab[i]处于MOVED状态，则当前线程也帮忙move（后面有介绍helpTransfer方法）
            tab = helpTransfer(tab, f);
        else {
        	// 考虑完上面的情况就可以插入
            V oldVal = null;
            // synchronized通过代码块，f对象锁
            synchronized (f) {
            	// f是之前保存的tab[i]值，判断是否修改了
                if (tabAt(tab, i) == f) {
                    if (fh >= 0) {
                    	// 遍历tab[i]这个链表
                        binCount = 1;
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            // 如果key相等（容器中已经含有key）
                            if (e.hash == hash && ((ek = e.key) == key || (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                // onlyIfAbsent = false就要更新（在前面已经介绍过这个参数了）
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            Node<K,V> pred = e;
                            // 到达链表的尾端，则只能插入到尾端
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key,value, null);
                                break;
                            }
                        }
                    }
                    else if (f instanceof TreeBin) {
                    	// 此hash桶还可能是红黑树实现
                        Node<K,V> p;
                        // 由于已经是红黑树实现，插入节点只会增加数量，binCount记录hash桶中的元素个数已经没有意义了
                        binCount = 2;
                        // 调用红黑树中的putTreeVal方法插入/更新节点（与此方法类似）
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key, value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }
            // 如果binCount > TREEIFY_THRESHOLD(前面介绍过该属性，链表转红黑树阈值，默认是8)
            if (binCount != 0) {
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);
                if (oldVal != null)
            		// 说明只是更新操作，提前返回
                    return oldVal;
                break;
            }
        }
    }
    // 计数
    addCount(1L, binCount);
    return null;
}
```
### ③、`putAll`方法
&emsp;&emsp;`putAll`方法，复制其它`map`容器中的所有`key-value`到本容器，调用`putVal`方法。
```java
/**
 * 复制容器m中的所有key-value到本容器，调用putVal方法（在前面介绍的最后一个构造器中使用了）
 */
public void putAll(Map<? extends K, ? extends V> m) {
	// 复制前需要判断是否容得下（扩容），后面有介绍此方法
    tryPresize(m.size());
    for (Map.Entry<? extends K, ? extends V> e : m.entrySet())
    	// 前面介绍了putVal方法
        putVal(e.getKey(), e.getValue(), false);
}
```
## 5、`get`方法
&emsp;&emsp;`get`方法，通过`key`查找`value`。
```java
public V get(Object key) {
    Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
    // 调用spread方法计算hash值（下文有介绍此方法）
    int h = spread(key.hashCode());
    if ((tab = table) != null && (n = tab.length) > 0 && (e = tabAt(tab, (n - 1) & h)) != null) {
        if ((eh = e.hash) == h) {
        	// 此种情况hash桶是链表实现，判断表头的key是否是要找的
            if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                return e.val;
        }
        else if (eh < 0)
        	// tab[(n - 1) & h].hash < 0 此时表示正在扩容，调用find查找
            return (p = e.find(h, key)) != null ? p.val : null;
        // 红黑树的情况已经返回，只能是链表了，并且表头判断了不是，遍历其它剩下节点即可
        while ((e = e.next) != null) {
            if (e.hash == h &&
                ((ek = e.key) == key || (ek != null && key.equals(ek))))
                return e.val;
        }
    }
    return null;
}
```

## 6、`hash`计算方法
&emsp;&emsp;在JDK1.8的版本中计算hash的方法为`spread`。方法调用示例`int h = spread(key.hashCode());`。

```java
/**
 * 入口参数h一般为某个对象的hashCode（此方法在Object类中有定义）
 */
static final int spread(int h) {
	// 将 h的低 16 位 与 高 16 位 异或
	// HASH_BITS是一个常量 0x7fffffff（32位有符号int型的最大值）
	// & HASH_BITS,主要是将前面异或的结果最高位置零，int是有符号整形，计算在table中的index时要求余，所以不能出现负数
    return (h ^ (h >>> 16)) & HASH_BITS;
}
```
## 7、链表、红黑树转换方法
&emsp;&emsp;前面介绍了`TREEIFY_THRESHOLD`常量表示链表长度超过该值就要转成红黑树，`UNTREEIFY_THRESHOLD`常量表示红黑树中节点不大于该值，就需要转换会链表。下面附上链表、红黑树转换方法。
### ①、`treeifyBin`方法
&emsp;&emsp;此方法是将hash桶的链表实现转为红黑树。
```java
/**
 * 将tab[index]的hash桶由链表转换成红黑树
 */
private final void treeifyBin(Node<K,V>[] tab, int index) {
    Node<K,V> b; int n, sc;
    if (tab != null) {
    	// 转换前还要满足一条件，tab.length > MIN_TREEIFY_CAPACITY(前面介绍过这个常量)
    	// 这样做的目的是防止tab数组太短而造成链表太长（hash冲突较多）的情况
        if ((n = tab.length) < MIN_TREEIFY_CAPACITY)
        	// 尝试扩容为原来的2倍
            tryPresize(n << 1);
        else if ((b = tabAt(tab, index)) != null && b.hash >= 0) {
            synchronized (b) {
            	// 修改tab[index]肯定需要把这个hash桶锁起来（与Hashtable相比，后者锁整个容器，ConcurrentHashMap减少了锁的粒度）
                if (tabAt(tab, index) == b) {
                    TreeNode<K,V> hd = null, tl = null;
                    // 下面遍历该链表，把链表中的所有节点Node转换成TreeNode对象，并且串成链表
                    for (Node<K,V> e = b; e != null; e = e.next) {
                        TreeNode<K,V> p = new TreeNode<K,V>(e.hash, e.key, e.val, null, null);
                        if ((p.prev = tl) == null)
                            hd = p;
                        else
                            tl.next = p;
                        tl = p;
                    }
                    // 这里调用TreeBin构造方法生成红黑树，并且赋值给tab[index]即转成了红黑树
                    setTabAt(tab, index, new TreeBin<K,V>(hd));
                }
            }
        }
    }
}
```
### ①、`untreeify`方法
&emsp;&emsp;此方法是将hash桶的红黑树实现转为链表。
```java
static <K,V> Node<K,V> untreeify(Node<K,V> b) {
    Node<K,V> hd = null, tl = null;
    // 遍历红黑树，将节点全部转换哼Node对象，并且串成链表，返回即可
    for (Node<K,V> q = b; q != null; q = q.next) {
        Node<K,V> p = new Node<K,V>(q.hash, q.key, q.val, null);
        if (tl == null)
            hd = p;
        else
            tl.next = p;
        tl = p;
    }
    return hd;
}
```

## 8、扩容相关的方法
### ①、`resizeStamp`方法
&emsp;&emsp;`resizeStamp`方法的返回值为`高16位置0，第16位为1，低15位存放当前容量m，代表的2^m == n`
```java
static final int resizeStamp(int n) {
	// Integer.numberOfLeadingZeros(n)用于计算n转换成二进制后前面有几个0
	// RESIZE_STAMP_BITS = 16常量，也就是(1 << (RESIZE_STAMP_BITS - 1)结果为 0000 0000 0000 0000 1000 0000 0000 0000
	// resizeStamp的返回值高16位置0，第16位为1，低15位存放当前容量m，代表的2^m == n
    return Integer.numberOfLeadingZeros(n) | (1 << (RESIZE_STAMP_BITS - 1));
}
```
### ②、`tryPresize`方法
&emsp;&emsp;`tryPresize`方法的作用是尝试扩容容器，使之能接受`size`个`key-value`。
```java
/**
 * 尝试扩容当前容器，使能接收size个key-value
 * 此方法在putAll（复制其它容器）、treeifyBin链表转红黑树有调用（在对应的方法中标记了，可以去前面看看）
 */
private final void tryPresize(int size) {
	// size是预计存放的key-value数，但是由于负载因子（默认0.75）的存在，所以实际上还需要扩大，
	// tableSizeFor(n)计算不小于n的最小2的次幂,比如n = 12时，返回16=2^4
    int c = (size >= (MAXIMUM_CAPACITY >>> 1)) ? MAXIMUM_CAPACITY : tableSizeFor(size + (size >>> 1) + 1);
    int sc;
    // 翻翻前面介绍sizeCtl属性吧，>= 0表示 刚构造完 获取 初始化完
    while ((sc = sizeCtl) >= 0) {
        Node<K,V>[] tab = table; int n;
        if (tab == null || (n = tab.length) == 0) {
        	// 此种情况是tab没有初始化，与initTable方法（前面已介绍过了）几乎一毛一样的
            n = (sc > c) ? sc : c;
            // UnSafe类，将SIZECTL（可理解为sizeCtl的内存地址）修改为-1
            // 修改前判断sizeCtl是否发生了修改，sc之前复制sizeCtl的值
            if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                try {
                    if (table == tab) {
                        @SuppressWarnings("unchecked")
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        table = nt;
                        // n >>> 2的结果是n / 4，因为默认负载因子是0.75
                        // sc保存key-value的阈值
                        sc = n - (n >>> 2);
                    }
                } finally {
                	// 初始化后sizeCtl保存为sc(前面已经赋值tab.length * 负载因子)
                    sizeCtl = sc;
                }
            }
        }
        else if (c <= sc || n >= MAXIMUM_CAPACITY)
        	// tab 已经判断了非null，则sc(sizeCtl)存储的是key-value的阈值，超过需要的容量c，此时不用扩容，同样也不用移动key-value
            break;
        else if (tab == table) {
        	// 前面已经扩容了table容量能得下size个key-value
        	// 现在需要考虑移动各个hash桶中的key-value
            int rs = resizeStamp(n);
            if (sc < 0) {
                Node<K,V>[] nt;
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || (nt = nextTable) == null || transferIndex <= 0)
                    break;
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                    transfer(tab, nt);
            } else if (U.compareAndSwapInt(this, SIZECTL, sc, (rs << RESIZE_STAMP_SHIFT) + 2))
                transfer(tab, null);
        }
    }
}
```
### ③、`helpTransfer`方法
&emsp;&emsp;`helpTransfer`方法的作用是在有其它线程在扩容，当前容器尝试去辅助移动`key-value`，缩短扩容的时间。
```java
/**
 * Helps transfer if a resize is in progress.
 */
final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
    Node<K,V>[] nextTab; int sc;
    // 在transfer方法中，我们把table[i]置为ForwardingNode对象，说明该hash桶正在移动
    // 移动完后nextTab == null（在transfer方法中有设置）
    if (tab != null && (f instanceof ForwardingNode) && (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
        int rs = resizeStamp(tab.length);
        while (nextTab == nextTable && table == tab && (sc = sizeCtl) < 0) {
            if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                sc == rs + MAX_RESIZERS || transferIndex <= 0)
                break;
            // 每有一个线程来帮助迁移，sizeCtl就+1，初始值为(rs << RESIZE_STAMP_SHIFT) + 2)（在tryPresize()设置），之后再transfer中会用到
            if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {
                transfer(tab, nextTab);
                break;
            }
        }
        return nextTab;
    }
    return table;
}
```
### ④、`transfer`方法
&emsp;&emsp;`ConcurrentHashMap`支持并发扩容，`transfer`是容器中的扩容方法。
```java
/**
 * Moves and/or copies the nodes in each bin to new table. See
 * above for explanation.
 */
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
    int n = tab.length, stride;
    // stride记录并发量，NCPU常量记录CPU的数
    if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
        stride = MIN_TRANSFER_STRIDE; // subdivide range
    if (nextTab == null) {
    	// 初始化nextTab
        try {
            @SuppressWarnings("unchecked")
            // n << 1相当于 n * 2,也就是扩大为原来的2倍
            Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
            nextTab = nt;
        } catch (Throwable ex) {      // try to cope with OOME
            sizeCtl = Integer.MAX_VALUE;
            return;
        }
        // nextTable是实例属性，用于记录正在扩容时暂时存放（前面介绍属性说过）
        nextTable = nextTab;
        // [0, transferIndex)的hash桶中的key-value待移动nextTab，>=transferIndex的桶都已分配出去
        transferIndex = n;
    }
    int nextn = nextTab.length;
    // 扩容时的特殊节点，标明此节点正在进行迁移，扩容期间的元素查找要调用其find()方法在nextTable中查找元素。
    ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
    // 当前线程是否需要继续寻找下一个可处理的节点
    boolean advance = true;
    // 所有桶是否已经完成迁移
    boolean finishing = false;
    for (int i = 0, bound = 0;;) {
        Node<K,V> f; int fh;
        // 此循环的作用是确定当前线程要迁移的桶的范围或通过更新i的值确定当前范围内下一个要处理的节点。
        while (advance) {
            int nextIndex, nextBound;
            if (--i >= bound || finishing)
                advance = false;
            else if ((nextIndex = transferIndex) <= 0) {
            	// 当前已经迁移完毕
                i = -1;
                advance = false;
            }
            else if (U.compareAndSwapInt(this, TRANSFERINDEX, nextIndex, nextBound = (nextIndex > stride ? nextIndex - stride : 0))) {
            	// 确定当前线程每次分配的待迁移桶的范围为[bound, nextIndex)
                bound = nextBound;
                i = nextIndex - 1;
                advance = false;
            }
        }
        // tab的长为n，需要移动的桶范围[0, n)
        if (i < 0 || i >= n || i + n >= nextn) {
            int sc;
            // 扩容操作全部完成，赋值table = nextTab，清空nextTable（临时指向新表）
            if (finishing) {
                nextTable = null;
                table = nextTab;
                // sizeCtl重新恢复为 nextTab.length * 0.75(即2*n - n / 2)
                sizeCtl = (n << 1) - (n >>> 1);
                return;
            }
            // 当前线程已经完成移动任务，sizeCtl - 1
            if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                    return;
                finishing = advance = true;
                // 这里又赋值i = n，主要是在上面的while循环中再次检查是否还有没移动的hash桶
                i = n;
            }
        }
        else if ((f = tabAt(tab, i)) == null)
        	// tabAt[i] == null,将它置为fwd
            advance = casTabAt(tab, i, null, fwd);
        else if ((fh = f.hash) == MOVED)
        	// MOVED是一个常量，标记已经移动完毕了
            advance = true;
        else {
            synchronized (f) {
            	// 移动f 即tab[i]中的元素，给f加上锁（Hashtable是锁上整个对象，这样减少锁的粒度，故效率高）
                if (tabAt(tab, i) == f) {
                	// 当前hash桶拆分成两个hash桶（因为整个容器扩大为原来2倍）
                    Node<K,V> ln, hn;
                    if (fh >= 0) {
                    	// fh >= 0，表示该hash桶是链表实现
						
						// （关键）此处的n是扩展前的table大小（并且n是2的次幂），& n 结果是判断 二进制中这一位是否为1，比如n = 16 =  0001 0000
						// 任何数与n与操作，要不是得到 0001 0000，要不就是 0000 0000
                        int runBit = fh & n;
                        
                        // 下面这段代码与Hashtable中的扩容是一样的（那篇博客扩容讲的比较清晰，可以看看），主要的思想是拆分成两个链表，
                        // 原始有16个hash桶，并且当前扩展的桶存放的是余数为10的元素，则放入该桶的hash值有 10、26、42、58...
                        // 现在扩展为原来的2倍，32个桶，则当前桶下的元素 分别移动到 hash % 32 == 10 和 hash % 32 = 10 + 16两个桶下
                        // 比如10、42放入第一个桶，26、48放入第二个桶
                        // 其实要判断的就是 hash & 2n 是否大于 n
                        
                        // 首先取出链表的尾端节点，更新runBit为尾节点的hash值与n的二进制形式与操作后对应的为是否为1
                        Node<K,V> lastRun = f;
                        for (Node<K,V> p = f.next; p != null; p = p.next) {
                            // 与runBit一样，此处同样计算的是节点p的hash值二进制形式中对应位是否是1
                            int b = p.hash & n;
                            if (b != runBit) {
                            	// 更新runBit，为p.hash & n（为啥要用if判断，b == runBit 如果相等就不用更新）
                                runBit = b;
                                lastRun = p;
                            }
                        }
                        // 判断尾端节点放到哪个链表下，前面更新runBit
                        // runBit == 0,说明p.hash % 2n < n
                        if (runBit == 0) {
                            ln = lastRun;
                            hn = null;
                        }
                        else {
                            hn = lastRun;
                            ln = null;
                        }
                        // 然后将链表其它节点也串接到两个链表上
                        for (Node<K,V> p = f; p != lastRun; p = p.next) {
                            int ph = p.hash; K pk = p.key; V pv = p.val;
                            if ((ph & n) == 0)
                                ln = new Node<K,V>(ph, pk, pv, ln);
                            else
                                hn = new Node<K,V>(ph, pk, pv, hn);
                        }
                        // ln链表 hash % 2n < n
                        setTabAt(nextTab, i, ln);
                        // ln链表 hash % 2n > n
                        setTabAt(nextTab, i + n, hn);
                        setTabAt(tab, i, fwd);
                        advance = true;
                    }
                    else if (f instanceof TreeBin) {
                    	// 剩下的就是红黑树实现了
                        TreeBin<K,V> t = (TreeBin<K,V>)f;
                        TreeNode<K,V> lo = null, loTail = null;
                        TreeNode<K,V> hi = null, hiTail = null;
                        int lc = 0, hc = 0;
                        // 遍历红黑树，同上拆分成两个链表，判断条件(h & n) == 0（其实就是判断 h % 2n > n）
                        for (Node<K,V> e = t.first; e != null; e = e.next) {
                            int h = e.hash;
                            TreeNode<K,V> p = new TreeNode<K,V>(h, e.key, e.val, null, null);
                            if ((h & n) == 0) {
                                if ((p.prev = loTail) == null)
                                    lo = p;
                                else
                                    loTail.next = p;
                                loTail = p;
                                ++lc;
                            }
                            else {
                                if ((p.prev = hiTail) == null)
                                    hi = p;
                                else
                                    hiTail.next = p;
                                hiTail = p;
                                ++hc;
                            }
                        }
                        // 如果链表的长度超过UNTREEIFY_THRESHOLD阈值，则转换成红黑树
                        ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :(hc != 0) ? new TreeBin<K,V>(lo) : t;
                        hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :(lc != 0) ? new TreeBin<K,V>(hi) : t;
                        // 然后分别放入nextTab[i],nextTab[i + n]
                        setTabAt(nextTab, i, ln);
                        setTabAt(nextTab, i + n, hn);
                        setTabAt(tab, i, fwd);
                        advance = true;
                    }
                }
            }
        }
    }
}
```

# 四、总结
&emsp;&emsp;总的来说JDK1.7版本的`ConcurrentHashMap`相当于多个`Hashtable`容器（其实就是段的概念），使用多个`RententLock`锁，当要修改某个段中的元素时，只要锁住对应的段即可（分段锁），这样相比于`Hashtable`容器锁住整个容器，减少了锁的粒度，大大提高了并发效率。
&emsp;&emsp;在JDK1.8版本的`ConcurrentHashMap`中，去除了段的概念，但它仍然高效的原因是进行插入、删除等结构性调整时锁hash桶（table数组中的元素），并且在扩容时，其他读写线程可以帮助正在扩容的线程一起进行扩容，同样减少了锁的粒度。
&emsp;&emsp;这篇博客真的写了好几天，`ConcurrentHashMap`的源码的相关方法也不知道看了多少遍了，但是博客可能存在不少理解错误，欢迎在评论区留言讨论~

==（我有一句话不知该不该说，Oracle花大把精力更新Java 1x 啥的，还不如把源码可读性弄好点...）==

资料参考：
https://blog.csdn.net/tp7309/article/details/76532366
https://blog.csdn.net/xingxiupaioxue/article/details/88062163

