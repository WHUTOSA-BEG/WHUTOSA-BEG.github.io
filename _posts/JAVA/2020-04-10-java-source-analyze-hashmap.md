---
layout: post
title: Java容器之HashMap源码分析
excerpt: 妈妈再也不用担心我不懂HashMap了
date: 2020-04-10
author: hestyle
authorSite: https://blog.csdn.net/qq_41855420
firstPostShow: hestyle's blog
firstPostURL: https://blog.csdn.net/qq_41855420/article/details/105425716
color: rgb(1,63,148)
cover: /assets/post/java.jpg
tags: Java
---

&emsp;&emsp;最近面试被问`HashMap`容器的实现原理，答的一塌糊涂。。。虽说一直念叨着说要看看`Java`容器的源码，但总是被耽搁了，今天终于静下心来看了🤦‍♂️。

&emsp;&emsp;**注明**：以下源码分析都是基于`jdk 1.8.0_221`版本
![](https://img-blog.csdnimg.cn/20200410090633414.png)

## 一、`HashMap`概述（==一图以蔽之==）
&emsp;&emsp;`HashMap`的类声明如下

```java
public class HashMap<K,V> extends AbstractMap<K,V>
    implements Map<K,V>, Cloneable, Serializable
```
&emsp;&emsp;`HashMap`是一个`<key,value>`（或称`键值对`）容器，其底层实现是使用一个`hash数组`指向多个不同的`链表`。每次我们放入一个`<key,value>`，它会自动计算key对应的`hash值`，然后根据`hash值`插入到不同的`链表`中。
![](https://img-blog.csdnimg.cn/20200410122114268.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQxODU1NDIw,size_16,color_FFFFFF,t_70)

&emsp;&emsp;**注明**：可能会有人对为啥要用`hash数组`套`链表`产生疑问，这是因为实际插入过程中会出现多个`<key,value>`的key计算出的`hash值`相同（==哈希冲突==），如上图的`table[1]`。但是当链表太长时，在容器中查找`<key,value>`，每次都要遍历耗时长，降低了查找效率，所以在`Java 8`中，引入了`红黑树`。默认当某个hash值下超过了8个`<key,value>`，此时就需要转化成`红黑树`，如果上图中的`table[14]`。

## 二、`HashMap`类的属性
### 1、`HashMap`类静态属性

```java
/**
 * 序列化的版本号
 */
private static final long serialVersionUID = 362498820763181265L;
/**
 * 默认的初始化容量大小，并且必须是2的幂（主要是考虑效率，后面有介绍）
 */
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16

/**
 * 最大的容量（容器中存放<key, value>的最大数量）
 */
static final int MAXIMUM_CAPACITY = 1 << 30;

/**
 * 负载因子
 * 当容器中<key, value>的数量超过capacity * DEFAULT_LOAD_FACTOR时，需要扩容
 */
static final float DEFAULT_LOAD_FACTOR = 0.75f;

/**
 * 链表转红黑树阈值
 * 当某个hash值下<key, value>用链表存储，并且链表长度不小于该值，就需要转成红黑树
 */
static final int TREEIFY_THRESHOLD = 8;

/**
 * 红黑树转链表阈值
 * 当某个hash值下<key, value>是用红黑树存储，并且树中的节点数小于该值，就需要转成链表
 */
static final int UNTREEIFY_THRESHOLD = 6;

/**
 * 最小树形化容量阈值
 * 当哈希表中的容量 > 该值时，才允许将链表转成红黑树操作，否则直接扩容。
 * 为了避免进行扩容、链表转红黑树选择的冲突，并且这个值不能小于 4 * TREEIFY_THRESHOLD（链表转红黑树阈值）
 */
static final int MIN_TREEIFY_CAPACITY = 64;
```
### 2、`HashMap`非静态属性
&emsp;&emsp;`transient`关键字的作用是在序列化的时候排除该属性，比如写入硬盘持久化，用这个关键字修饰的属性在对象保存时不会写入。（不过`HashMap`类在尾端重写了序列化方法，手动指定了需要序列化的属性）
```java
/**
 * table数组，也称hash桶数组
 */
transient Node<K,V>[] table;

/**
 * entrySet属性，把<K,V>存放到Set容器中（一般hashmap的遍历用此属性）
 */
transient Set<Map.Entry<K,V>> entrySet;

/**
 * 容器key-value数量（注意与容器的容量（容器可存放的数量）不同）
 */
transient int size;

/**
 * 容器进行结构性调整（增加或者删除键值对等操作，不包括修改value值）的次数
 */
transient int modCount;

/**
 * 容器中能容纳的key-value极限，capacity * loadFactor，超过就需要扩容
 */
int threshold;

/**
 * 负载因子，默认是0.75(前面类的静态属性已经定义过了)
 */
final float loadFactor;
```
$\color{red}注意：$上面提到的`容量`就是`table`数组的长度，`size`是容器中存放的`key-value`数量，`threshold` = 容量 * 负载因子，表示的该容器最多可以放置多少个`key-value`。
## 三、`HashMap`类的构造器
&emsp;&emsp;查看`HashMap`类文件，可以发现一共有4个构造器。

```java
/**
 * @param  initialCapacity 初始化容量大小
 * @param  loadFactor      负载因子
 * @throws IllegalArgumentException if the initial capacity is negative
 *         or the load factor is nonpositive
 */
public HashMap(int initialCapacity, float loadFactor) {
	// 检查initialCapacity的合法性
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal initial capacity: " + initialCapacity);
    // 检查initialCapacity是否超过了可设置的最大容量（类静态属性）
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    // 检查loadFactor负载因子的合法性
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal load factor: " +
                                           loadFactor);
    this.loadFactor = loadFactor;
    //初始化threshold稍微复杂一点，tableSizeFor方法解析见本博客尾端
    this.threshold = tableSizeFor(initialCapacity);
}

/**
 * @param  initialCapacity 初始化容量大小
 * @throws IllegalArgumentException if the initial capacity is negative.
 */
public HashMap(int initialCapacity) {
	//默认负载因子为0.75
    this(initialCapacity, DEFAULT_LOAD_FACTOR);
}

/**
 * 只设置负载因子为0.75，其它值全部默认
 */
public HashMap() {
    this.loadFactor = DEFAULT_LOAD_FACTOR;
}

/**
 * 复制构造函数，将另外一个map初始化构造
 *
 * @param   m 其它map容器
 * @throws  NullPointerException
 */
public HashMap(Map<? extends K, ? extends V> m) {
    this.loadFactor = DEFAULT_LOAD_FACTOR;
    //将m容器中的所有entry放入新建的容器对象中
    putMapEntries(m, false);
}
```
## 四、增加`key-value`相关方法
### 1、`put`方法
&emsp;&emsp;`put`方法，往容器中添加`key-value`，允许`key = null`，也允许`value = null`。
```java
/**
 * 往容器中添加`key-value`，允许`key = null`，也允许`value = null`
 */
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}
```

### 2、`putVal`方法
&emsp;&emsp;`putVal`方法的作用是往`map`容器中插入一个`key-value`。
```java
/**
 * Implements Map.put and related methods.
 *
 * @param hash key的hash值（调用hash()方法）
 * @param key 插入键值对key
 * @param value 插入键值对value
 * @param onlyIfAbsent 设为true时，表示如果容器已经存在这个key就不进行修改
 * @param evict 为 false时，表示容器正处于创建（其它map传入初始化）
 * @return previous value, or null if none
 * 
 */
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    // tab指向 对象的table数组（hash桶数组），p 指向hash对应的桶
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    // 如果容器为空，则需要调用resize方法，初始化table数组
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    // 将p指向hash对应的hash桶
    if ((p = tab[i = (n - 1) & hash]) == null)
    	// (n - 1) & hash求出hash对应的table数组下标，如果这个位置为空，说明这个桶为空
    	// 直接放入table中，不需要生成链表、红黑树等
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;
        // 根据p（指向对应的hash桶），找到key对应的节点位置
        if (p.hash == hash && ((k = p.key) == key || (key != null && key.equals(k))))
        	// 如果当前p指向桶第一个元素就是key
            e = p;
        else if (p instanceof TreeNode)
        	// 如果p指向的内容（hash桶对应的第一个元素）是红黑树的对象，说明该桶已转换为红黑树，调用putTreeVal插入
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
        	// 否则桶内实现是链表，只能遍历链表查找key
            for (int binCount = 0; ; ++binCount) {
            	// p.next == null。即链表的尾端
                if ((e = p.next) == null) {
                	// 还没找到，则需要插入节点
                    p.next = newNode(hash, key, value, null);
                    // 如果该桶的元素超过了 TREEIFY_THRESHOLD，需要进行扩容或者转成红黑树
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                //如果当前节点以及成功匹配key，退出
                if (e.hash == hash && ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        if (e != null) {
        	//找到了key对应的位置，再赋value
            V oldValue = e.value;
            // onlyIfAbsent入口参数，为true，则不更新value（前面已说明）
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            //成功更新了节点
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    //容器中的key-value数自增，并且判断是否需要扩容（前面已多次说明threshold = 容器容量 * 最大负载因子）
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```

### 3、`putMapEntries`方法
&emsp;&emsp;`putMapEntries`方法作用是将其他map容器中的`key-value`复制到本容器中。
```java
/**
 * 将一个map容器中的key-value复制到本容器
 *
 * @param m 其它map容器
 * @param evict 为 false时，表示容器正处于创建（其它map传入初始化）
 */
final void putMapEntries(Map<? extends K, ? extends V> m, boolean evict) {
    int s = m.size();
    //只有当m非空的时候才有遍历的必要
    if (s > 0) {
    	//如果table为空，则需要判断m中的元素个数是否会超过本容器可容纳的数量（容器容量 * 负载因子）
        if (table == null) { // pre-size
        	//由于table为空，我们需要将 m.size() / 负载因子loadFactor，得到需要的最小空间
            float ft = ((float)s / loadFactor) + 1.0F;
            int t = ((ft < (float)MAXIMUM_CAPACITY) ? (int)ft : MAXIMUM_CAPACITY);
            //最小空间都大于容器当前能存放的最大数量（threshold = 当前容量 * 负载因子）
            if (t > threshold)
            	//当前容器无法容纳，则需要计算不必t小的2的幂
                threshold = tableSizeFor(t);
        }
        else if (s > threshold)
        	// table ！= null，此时我们只要判断 m.size() > 容器当前能存放的最大数量（threshold = 当前容量 * 负载因子）,从而决定是否扩容
            resize();
        // 经过上面的扩容操作，已经保证 m.size() < 容器当前能存放的最大数量（threshold = 当前容量 * 负载因子）,遍历m放入本容器即可
        for (Map.Entry<? extends K, ? extends V> e : m.entrySet()) {
            K key = e.getKey();
            V value = e.getValue();
            putVal(hash(key), key, value, false, evict);
        }
    }
}
```

## 五、删除`key-value`相关方法
### 1、`remove`方法
&emsp;&emsp;`remove`方法是提供给外界的删除`key-value`的接口。
```java
/**
 * 根据key删除key-value，删除成功返回对应的value，否则返回null
 */
public V remove(Object key) {
    Node<K,V> e;
    //调用removeNode方法
    return (e = removeNode(hash(key), key, null, false, true)) == null ?
        null : e.value;
}
```

### 2、`removeNode`方法
&emsp;&emsp;`removeNode`方法是删除`key-value`的具体实现（注意不对外展示）。
```java
/**
 * Implements Map.remove and related methods.
 *
 * @param hash key对应的hash（调用hash()方法即可计算得到）
 * @param key 待删除的key-value对应的key
 * @param value key-value对应的value，只有当matchValue == true时，此参数才有意义
 * @param matchValue 如果设为true，删除的时候还需要匹配value才能删
 * @param movable 设为false，表示删除成功了不移动其它节点（一般设为true，即删除节点后需要进行调整）
 * @return 删除成功则返回key对应的value，否则返null
 */
final Node<K,V> removeNode(int hash, Object key, Object value, boolean matchValue, boolean movable) {
	// tab用于指向table数组（hash桶数组），p用于指向传入的key对应的hash所对应的hash桶
    Node<K,V>[] tab; Node<K,V> p; int n, index;
    // 如果table数组存在hash对应的桶
    if ((tab = table) != null && (n = tab.length) > 0 && (p = tab[index = (n - 1) & hash]) != null) {
    	// node的作用是指向key对应的节点位置
        Node<K,V> node = null, e; K k; V v;
        // 判断hash桶的第一个元素的key是否是匹配成功
        if (p.hash == hash && ((k = p.key) == key || (key != null && key.equals(k))))
            node = p;
        else if ((e = p.next) != null) {
        	// 匹配不成功，则需要判断这个桶是链表还是红黑树实现
            if (p instanceof TreeNode)
            	// 如果是 红黑树则调用getTreeNode方法
                node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
            else {
            	// 否则只能遍历链表
                do {
                    if (e.hash == hash && ((k = e.key) == key || (key != null && key.equals(k)))) {
                        node = e;
                        break;
                    }
                    p = e;
                } while ((e = e.next) != null);
            }
        }
        // 如果node != null，则说明容器中存在key对应的key-value
        // 如果 matchValue == true，则还需要匹配value，才能删
        if (node != null && (!matchValue || (v = node.value) == value || (value != null && value.equals(v)))) {
            // 如果node指向的对象是TreeNode类型，则调用红黑树对应的remove方法即可
            if (node instanceof TreeNode)
                ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
            else if (node == p)
            	// 如果node是table数组中的元素（hash桶内第一个元素）
                tab[index] = node.next;
            else
            	// 否则node是链表中的其他节点
                p.next = node.next;
            // 删除节点是结构性调整
            ++modCount;
            --size;
            afterNodeRemoval(node);
            return node;
        }
    }
    return null;
}
```
## 六、查找`key-value`相关方法
### 1、`get`方法
&emsp;&emsp;`get`方法的作用是根据`key`查找`value`。
```java
/**
 * 根据`key`查找`value`。
 * @see #put(Object, Object)
 */
public V get(Object key) {
    Node<K,V> e;
    //调用getNode方法查找
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}
```

### 2、`getNode`方法
&emsp;&emsp;`getNode`方法的作用是根据`key`查找查找`<key, value>`。
```java
/**
 * 根据`key`查找查找`<key, value>`
 *
 * @param hash key对应的hash值（调用hash()方法即可获取）
 * @param key the key
 * @return the node, or null if none
 */
final Node<K,V> getNode(int hash, Object key) {
	// tab用于指向table数组（hash桶数组），first用于指向hash桶第一个元素，e用于指向hash、key匹配到的节点位置
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    //查找table数组（hash桶数组）中是否存在hash对应的桶
    if ((tab = table) != null && (n = tab.length) > 0 && (first = tab[(n - 1) & hash]) != null) {
        //判断桶中的第一个元素first的key与需要查找的key是否想相同
        if (first.hash == hash && ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        //这个桶不能只有一个元素， > 1才有继续寻找的必要
        if ((e = first.next) != null) {
            if (first instanceof TreeNode)
            	//如果该hash桶是红黑树实现，调用红黑树对应的查找方法
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            //否则该hash桶是链表实现，遍历链表即可
            do {
                if (e.hash == hash && ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}
```

### 3、`containsKey`方法
&emsp;&emsp;`containsKey`方法的作用是判断容器中是否存在`key`对应的`key-value`。
```java
/**
 * 判断容器中是否存在`key`对应的`key-value`
 */
public boolean containsKey(Object key) {
	// 调用getNode方法，如果查找到了key-value则说明存在
    return getNode(hash(key), key) != null;
}
```

### 4、`containsValue`方法
&emsp;&emsp;`containsValue`方法的作用是判断容器中是否存在`value`对应的`key-value`。
```java
/**
 * 判断容器中是否存在`value`对应的`key-value`
 */
public boolean containsValue(Object value) {
	// tab用于指向table数组（hash桶数组）
    Node<K,V>[] tab; V v;
    if ((tab = table) != null && size > 0) {
    	//遍历table数组（hash桶数组）
        for (int i = 0; i < tab.length; ++i) {
        	// 遍历hash桶中的每一个元素
            for (Node<K,V> e = tab[i]; e != null; e = e.next) {
                if ((v = e.value) == value ||
                    (value != null && value.equals(v)))
                    return true;
            }
        }
    }
    return false;
}
```

## 七、其它方法
### 1、`tableSizeFor`方法
&emsp;&emsp;该方法用于找出不小于cap的最小2的幂。
&emsp;&emsp;假设cap 的二进制形式为`01xxxx...xxx`，先考虑n = cap的情况.
|n| n >>> x | n \| = n >>> x |
|--|--|--|
| 初始化为cap = `01xxxx...xxx` | n >>> 1结果 `001xxx...xxx` | `011xxx...xxx` |
| `011xxx...xxx` |n >>> 2结果 `00011x...xxx` | `01111x...xxx` |
| `01111x...xxx` |n >>> 4结果 `000001111xxx` | `011111111...` |

&emsp;&emsp;==表格所表达的意思就是依次将cap最高位为1后面的所有位都置为1==，第一次右移一位，`n |= n >>> 1`得到了两个1，第二次`n |= n >>> 2`，右移两位，得到了4个1，然后右移4位，得到了8个1...
&emsp;&emsp;然后`n += 1`,也就是二进制`011...111`进位`100...000`，正好是2的幂。
```java
/**
 * 返回不小于cap的最小的2的幂，比如cap == 3时，返回4，cap == 12时返回16等等
 */
static final int tableSizeFor(int cap) {
	// n = cap - 1是为了防止当cap本身就是2的幂，此时计算出的结果偏大了
	// 比如cap = 16（二进制”10000“），通过计算求得的n = "11111”，
	// 然后n + 1 = "100000" = 32, 实际上cap自身16就是解
    int n = cap - 1;
    // >>> 运算符的作用是 无符号右移（左边填充0），比如 ’11111‘ >>> 1的结果为'01111'
    // 如果你知道原码、补码的相关知识，这点很容易理解，不知道就先记着吧
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    //最后返回n + 1、MAXIMUM_CAPACITY（HashMap容器容量最大值）的较小值，
    //由于n计算时可能发生了溢出，所以需要判断是否小于0
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```
&emsp;&emsp;如果你还是没看懂这个解释，可以随便带入几个`cap`，比较计算后的二进制结果就会发现这个方法的作用。
### 2、`hash`方法（重要）
&emsp;&emsp;此方法用来计算`key`对象的`hash值`，从而决定放到`table表`（`hash桶`）的哪个表项中。
```java
/**
 * 计算key对象的hash值
 */
static final int hash(Object key) {
    int h;
    // hashCode是Object类的方法（一般重写tostring方法会让你重写这个方法）
    // 在HashMap容器中，hash值 == key.hashCode()的前16位 异或 key.hashCode()的后16位（主要是防止hash冲突，多个key的hash值一样）
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

### 3、`resize`方法（重要）
&emsp;&emsp;`resize`方法的作用是扩容。
```java
/**
 * Initializes or doubles table size.  If null, allocates in
 * accord with initial capacity target held in field threshold.
 * Otherwise, because we are using power-of-two expansion, the
 * elements from each bin must either stay at same index, or move
 * with a power of two offset in the new table.
 *
 * @return the table
 */
final Node<K,V>[] resize() {
	// 记录扩容前的状态
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    // 如果oldCap > 0，说明不是初始化
    if (oldCap > 0) {
    	// 如果oldCap 不小于HashMap容器定义的最大容量，修改threshold为最大值
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        // oldCap * 2后是否超过 MAXIMUM_CAPACITY
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY && oldCap >= DEFAULT_INITIAL_CAPACITY)
            // 容量变为原来的2倍，可存放的阈值（最大容量 * 负载因子）也 * 2，
            // 由于容量 * 2，所以阈值也需要 * 2
            newThr = oldThr << 1; // double threshold
    }
    else if (oldThr > 0)
    	// 根据前面判断知oldCap <= 0，此时时调用了HashMap的带参构造器，初始容量用threshold替换，
        //在带参构造器中，threshold的值为 tableSizeFor() 的返回值，也就是2的幂，而不是 capacity * load factor
        newCap = oldThr;
    else {
    	// 根据前面两个判断，oldCap <= 0 且 oldThr > 0，即调用了默认构造器
    	// 此时容器容量 newCap 赋值默认初始化容量，
    	// 容器最大存放数量newThr 赋值 默认负载因子 * 默认初始化容量
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    if (newThr == 0) {
    	//重新计算阈值
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ? (int)ft : Integer.MAX_VALUE);
    }
    // 更新当前容器可存放的最大数量
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
    // 创建新的 table数组（hash桶数组）
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    if (oldTab != null) {
    	//将oldTab中的所有key-value复制到newTab，遍历oldTab数组（hash桶数组）
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            //当前hash桶不为空才有遍历的必要
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                // 如果该hash桶中国只有一个元素，直接复制
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                // 如果该hash桶是红黑树实现，调用split方法复制
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else {
                	// 否则该hash桶是链表实现，需要遍历链表
                	// 将源链表拆分根据成两个链表，原链表中的所有节点(Node.hash % oldCap) == 0
                	// loHead、loTail指向第一个链表的头、尾，链表中的(Node.hash & oldCap) == 0
                	// hiHead、hiTail指向第二个链表的头、尾，链表中的(Node.hash & oldCap) ！= 0
                    
                    // 比如 oldCap = 16时，hash = 13，29，45，61...都应该放在oldTab[12]这个桶下
                    // 先应 newCap = 2 * oldCap = 32，需要拆分成newTab[12]、newTab[12 + oldCap = 28]两个桶
                    // newTab[12] = [13, 45],  hash % newCap = hash % 32 = 13
                    // newTab[12 + oldCap = 28] = [29, 61], hash % newCap = hash % 32 = 28
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;
                        if ((e.hash & oldCap) == 0) {
                        	//放到第一个链表
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        else {
                        	//放到第二个链表
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    //然后分别将第一个链表放入newTab[j]
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    //第二个链表放入newTab[j + oldCap]
                    if (hiTail != null) {
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
### 4、`treeifyBin`方法（重要）
&emsp;&emsp;`treeifyBin`方法是将某个链表实现的`hash桶`转换为红黑树。
```java
/**
 * @parm hash 代转换成红黑树的hash桶对应的hash值
 */
final void treeifyBin(Node<K,V>[] tab, int hash) {
    int n, index; Node<K,V> e;
    // 如果容器的（长度）容量小于 MIN_TREEIFY_CAPACITY，则直接扩容
    if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
        resize();
    // 否则 hash对应的 桶不为空时，此时进行链表转红黑树操作
    else if ((e = tab[index = (n - 1) & hash]) != null) {
    	// (n - 1) & hash的作用是获取hash桶对应的下标（table数组），效果等同于 hash % n(n 是 tab数组的长度)，
    	// 这是由于n 是 2 的次幂，这也是为什么table的容量（长度）必须初始化为2 的次幂，简化求余操作
        TreeNode<K,V> hd = null, tl = null;
        // 遍历 tab[(n - 1) & hash]，将所有节点都转成 TreeNode 节点
        do {
        	// 将当前节点转换为 TreeNode 节点（注意此处并没转成红黑树）
            TreeNode<K,V> p = replacementTreeNode(e, null);
            if (tl == null)
                hd = p;
            else {
                p.prev = tl;
                tl.next = p;
            }
            tl = p;
        } while ((e = e.next) != null);
        // 然后在调用treeify方法，将hd链表转成红黑树
        if ((tab[index] = hd) != null)
            hd.treeify(tab);
    }
}
```

## 八、总结(一图以蔽之)

![](https://img-blog.csdnimg.cn/20200410194144148.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQxODU1NDIw,size_16,color_FFFFFF,t_70)
&emsp;&emsp;`HashMap`由一个`table数组`（`hash桶数组`）和若干个链表组成，当某个`hash桶`内的数量过多时（链表太长，查找效率低），此时需要将链表转结构成红黑树结构，默认是链表长度超过8就要转换（当然如果红黑树中的节点太少，默认是 < 6时，需要转换回链表结构）。每次插入、删除节点，只要维持`table数组`、各个链表、红黑树即可。


**已更新 [Java容器之Hashtable源码分析](https://hestyle.blog.csdn.net/article/details/105448273)，各位小伙伴可以对比着看，效果更佳哦~**
