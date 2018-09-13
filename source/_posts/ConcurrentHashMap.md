---
title: ConcurrentHashMap
images: /images/摘要配图/
date: 2018-09-10 00:54:16
tags:
---


#HashMap和ConcurrentHashMap的原理，非常重要哦！！！
最近在看JAVA并发编程实践，ConcurrentMap是Concurrent包中一个非常重要的同步容器，在工作中也会使用到它。对于这样的容器，想要最为一个合格的后端攻城狮，必须经常的将它的源码翻出来看看，不然搬砖就搬的太失败了。本文想记录下自己对HashMap和ConcurrentHashMap的分析。
在1.7版本和1.8版本中java的源码发生了巨大的变化，好多之前用ReenTrantLock的貌似都改成了使用CAS或者violatile来实现。只能说HB法则太好用了。。。。
## 1.7版本的HashMap
HashMap是key-value的数据结构，同时key和value均可以为null，如果key为null，默认是放到table[0]的位置。
HashMap的底层是**数组+链表**的结构。
![](http://pbhb4py13.bkt.clouddn.com/2018-09-10-15365906343289.png)
HashMap里面是存放了一个数组，数组的每个元素都是一个单向链表。链表的每个节点是Entry结构，成员结构为`key,value,hash,next`。
在HashMap中还有其他成员

1. capacity: 当前的数组容量，始终保持2^n， 扩容后大小为当前的2倍。
2. loadFactor: 负载因子，默认为0.75
3. threshold: 扩容的阈值，等于capacity * loadFactor

### put的过程
1.7版本的hashmap的实现比较简单。

```java
public V put(K key, V value) {
    // 当数组为null，初始化数组
    if (table == EMPTY_TABLE) {
        inflateTable(threshold);
    }
    // 如果 key 为 null，将这个key放在table[0]的位置
    if (key == null)
        return putForNullKey(value);
    // 1. 求 key 的 hash 值
    int hash = hash(key);
    // 2. 找到对应的数组下标
    int i = indexFor(hash, table.length);
    // 3. 遍历一下对应下标处的链表，看是否有重复的 key 已经存在，
    //    如果有，直接覆盖，put 方法返回旧值就结束了
    for (Entry<K,V> e = table[i]; e != null; e = e.next) {
        Object k;
        if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
            V oldValue = e.value;
            e.value = value;
            e.recordAccess(this);
            return oldValue;
        }
    }
 
    modCount++;
    // 4. 不存在重复的key，将entry加入到table的曹里面
    addEntry(hash, key, value, i);
    return null;
}
```

#### 数组的初始化
插入第一个元素时会初始化HashMap，主要是计算初始化的table大小和阈值，table大小必须是2^n

```java
private void inflateTable(int toSize) {
    // 保证数组大小一定是 2 的 n 次方。
    // 比如这样初始化：new HashMap(20)，那么处理成初始数组大小是 32
    int capacity = roundUpToPowerOf2(toSize);
    // 计算扩容阈值：capacity * loadFactor
    threshold = (int) Math.min(capacity * loadFactor, MAXIMUM_CAPACITY + 1);
    // 算是初始化数组吧
    table = new Entry[capacity];
    initHashSeedAsNeeded(capacity); 
}
```

#### 计算数组的具体位置

```java
static int indexFor(int hash, int length) {
    //hash对数组长度取模即可。
    return hash & (length-1);
}
```

#### 添加entry到链表中

```java
void addEntry(int hash, K key, V value, int bucketIndex) {
    // 如果当前 HashMap 大小已经达到了阈值，并且新值要插入的数组位置已经有元素了，那么要扩容
    if ((size >= threshold) && (null != table[bucketIndex])) {
        // 扩容
        resize(2 * table.length);
        // 扩容以后，重新计算 hash 值. 如果key为null，放到0的位置
        hash = (null != key) ? hash(key) : 0;
        // 重新计算扩容后的新的下标
        bucketIndex = indexFor(hash, table.length);
    }
    //
    createEntry(hash, key, value, bucketIndex);
}
// 这个很简单，其实就是将新值放到链表的表头，然后 size++
void createEntry(int hash, K key, V value, int bucketIndex) {
    //找到表头
    Entry<K,V> e = table[bucketIndex];
    //当前节点为新的表头，next指向e
    table[bucketIndex] = new Entry<>(hash, key, value, e);
    size++;
}
```

#### 数组扩容

```java
void resize(int newCapacity) {
    Entry[] oldTable = table;
    int oldCapacity = oldTable.length;
    if (oldCapacity == MAXIMUM_CAPACITY) {
        threshold = Integer.MAX_VALUE;
        return;
    }
    // 新的数组
    Entry[] newTable = new Entry[newCapacity];
    // 将原来数组中的值迁移到新的更大的数组中
    transfer(newTable, initHashSeedAsNeeded(newCapacity));
    table = newTable;
    threshold = (int)Math.min(newCapacity * loadFactor, MAXIMUM_CAPACITY + 1);
}
```
数组扩容会将原来的table[i]节点拆分到newTable[i]和newTable[i+oldLength]的位置。如果原来的数组长度为16，那么table[1]的节点的元素会分配到table[1]和table[17]的位置。

### get过程分析
get的过程比较简单

1. 根据key计算hash值
2. 根据hash值计算数组下标，hash & (length-1)
3. 遍历数组该位置的链表，直到找到key相等的key就可以了 **所以在同一个链表中，key的hash都是相同的。**

```java
public V get(Object key) {
    // key 为 null 的话，会被放到 table[0]，所以只要遍历下 table[0] 处的链表就可以了
    if (key == null)
        return getForNullKey();
    // 
    Entry<K,V> entry = getEntry(key);
 
    return null == entry ? null : entry.getValue();
}

final Entry<K,V> getEntry(Object key) {
    if (size == 0) {
        return null;
    }
 
    int hash = (key == null) ? 0 : hash(key);
    // 确定数组下标，然后从头开始遍历链表，直到找到为止
    for (Entry<K,V> e = table[indexFor(hash, table.length)];
         e != null;
         e = e.next) {
        Object k;
        if (e.hash == hash &&
            ((k = e.key) == key || (key != null && key.equals(k))))
            return e;
    }
    return null;
}
```

## 1.8版本的HashMap
相比于1.7版本的HashMap，主要有以下几个优化

1. Entry改成了Node，红黑树的节点为TreeNode
2. 当链表的长度大于8时，链表会转化成红黑树存储，提高查找效率。

![](http://pbhb4py13.bkt.clouddn.com/2018-09-11-15365960838429.png)

### Node结构
Node结构和Entry结构基本相同。使用红黑树时节点为TreeNode，可以根据第一个节点为Node还是TreeNode来判断是链表还是红黑树。

```java
static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        V value;
        Node<K,V> next;

        Node(int hash, K key, V value, Node<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
        }

        //重写了hashCode，key的hash^value的hash
        public final int hashCode() {
            return Objects.hashCode(key) ^ Objects.hashCode(value);
        }

        public final V setValue(V newValue) {
            V oldValue = value;
            value = newValue;
            return oldValue;
        }

        //判断两个Entry是不是相等，判断key和value都相等即可
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

#### put过程

```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
       //onlyIfAbsent表示只有key不存在才会进行put操作。
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        //table还没有初始化，要resize到16的初始大小
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        //根据hash找到具体的数据下表，赋值给p如果为null，直接在该下表new一个Node。
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        else {
            Node<K,V> e; K k;
            //hash相等，并且==或者equals判断key相等，使用==主要是判断null
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            //如果p是TreeNode结构，表示该链表是红黑树存储    
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                //单向链表存储结构 bitCount计数节点的个数
                for (int binCount = 0; ; ++binCount) {
                    //如果next节点为null，表示没有找到key相同的节点，那就新建一个Node，插入到最后面
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        //如果插入节点后，bitCount大于发直，要把该位置转化成红黑树的结构
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    //找到key相同的节点
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            //如果e不为null，onlyIfAbsent为false时，替换oldValue值。
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        if (++size > threshold)
            resize(); //扩容
        afterNodeInsertion(evict);
        return null;
    }
```

#### resize扩容过程

```java
final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
        int oldCap = (oldTab == null) ? 0 : oldTab.length; //容量，数组的长度
        int oldThr = threshold;
        int newCap, newThr = 0;
        if (oldCap > 0) {
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                newThr = oldThr << 1; // double threshold  扩大一倍的阈值，thr = cap*factor
        }
        else if (oldThr > 0) // initial capacity was placed in threshold
            newCap = oldThr;
        else {               // zero initial threshold signifies using defaults
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr;
        @SuppressWarnings({"rawtypes","unchecked"})
            Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap]; //创建新的数组
        table = newTab;
        if (oldTab != null) {
            for (int j = 0; j < oldCap; ++j) { //遍历老的数组
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null; //将老的table的位置设为null，便于垃圾回收
                    if (e.next == null) //如果只有当个元素
                        newTab[e.hash & (newCap - 1)] = e; //赋值给新的位置，并且位置不变
                    else if (e instanceof TreeNode)
                        //如果是红黑树，将红黑树的位置分成2份
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    else { // preserve order
                        //将链表赋值给新的数组，将hash值根据newCap的最高位分成2部分，保证老的曹位置均不需要改变，这样可以使用一套hash算法
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {
                            next = e.next;
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
                        //低位槽
                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                        //高位槽
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

#### 1.7resize过程死循环，1.8解决该问题
1.8在resize过程中出现了loTail和hiTail两个链表，这个地方有必要讲解下。在1.7版本采用的transfer在并发时会出现死循环，而1.8是按照原来链表的顺序，不会出现死循环的情况。画图讲解下把。

在1.7版本中，resize的过程是反着插入，如下图所示，比如链表是3->5 那么在新的链表插入会变成5->3。这样在多线程中会容易出现死循环。也是图中的AB线程所示。当resize中，A线程刚好执行到获取到链表的第一个Entry，设为e。这个时候线程A被剥夺CPU时间挂起，B线程执行put操作，那么B也会执行resize操作，当线程B执行完毕后就会在图中所示的newTable[3]->5->3。
这时候线程A会执行如下代码

```java
e.next = newTable[i];
newTable[i] = e;
e = e.next;
```
一顿操作后就会把3的引用又指向5了，这个时候A线程就会在这个链表死循环了。
![concurrenthashMap1](http://pbhb4py13.bkt.clouddn.com/2018-09-13-concurrenthashMap1.png)


1.8的resize过程中使用loHead和hiHead按照顺序移动新的链表，解决了死循环的问题。如下所示。
遍历原来的链表，使用loHead和loTail来记录头结点和尾节点，然后一直在尾节点添加新的节点，这样计算B线程执行完毕后，A线程再去执行时也是读取到和原来的一样的记录。不会造成死循环。
![concurrenthashmap3](http://pbhb4py13.bkt.clouddn.com/2018-09-13-concurrenthashmap3.png)


#### get过程分析
get的步骤和之前差不多

1. 判断key的hash值，根据hash & (length-1)找到下标
2. 如果key为null，找到table[0]的值，取出
3. 找到数组的位置，判断Node类型，如果是TreeNode，用红黑树的方法取数据
4. 遍历链表，找到key相等

```java
public V get(Object key) {
    Node<K,V> e;
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}
final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {
        // 判断第一个节点是不是就是需要的
        if (first.hash == hash && // always check first node
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        if ((e = first.next) != null) {
            // 判断是否是红黑树
            if (first instanceof TreeNode)
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
 
            // 链表遍历
            do {
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}
```

##1.7版本的ConcurrentHashMap
在1.7版本的ConcurrentHashMap中，为了提高并发的能力，采用了分段锁的概念。在内部采用了Segment的结构，一个Segment就是一个Hash Table结构，在Segment内部维护了链表数组。
![-c](http://pbhb4py13.bkt.clouddn.com/2018-09-12-15366839410846.jpg)
如上图所示，采用了Segment数组和HashEntry组成，也是数据+链表的实现方式。

### 成员结构
Segment是同步map的内部类，同步map由segment数组组成。

```java
/**
 * Segment 数组，存放数据时首先需要定位到具体的 Segment 中。
 */
final Segment<K,V>[] segments;
transient Set<K> keySet;
transient Set<Map.Entry<K,V>> entrySet;

static final class Segment<K,V> extends ReentrantLock implements Serializable {
    transient volatile int count;   //元素的个数  在插入时候，count++,在删除时候count--
    transient int modCount;       //对table造成影响的操作数目,在插入和删除数据的时候modcount++
    transient int threshold;     //容量
    transient volatile HashEntry<K,V>[] table;       //链表数组，每一个元素代表链表的头部
    final float loadFactor;
}
```

接着看HashEntry的组成，
将value设置为volatile类型的，在get的时候就不需要加锁，保证了可见性。

```java
    final K key;    //不可变， 
    final int hash;  //不可变
    volatile V value;   //可见
    final HashEntry<K,V> next;  //引用不可变
}
```
Segment的个数为2^n，方便使用位移操作加快定位的过程。key的hash值的高n位作为Segment的值，而低位作为HashEntry的定位。

### get操作

```java
//根据key获得segment的下表
public V get(Object key) {
    int hash = hash(key.hashCode());
    return segmentFor(hash).get(key, hash);
}
final Segment<K,V> segmentFor(int hash) { 
    return segments[(hash >>> segmentShift) & segmentMask];  //将hash右移segmentShift位后得到高n位，在跟segmentMask做&操作，得到高n位的值
}

//根据key和segment定位到table的表头。在
V get(Object key, int hash) {
    //count是volatile类型的，根据HB原则其他线程的put操作都被当前线程观察到。在其他线程的count++是在锁中完成、
    if (count != 0) { // read-volatile
        HashEntry<K,V> e = getFirst(hash);
        //玄幻遍历链表得到值
        while (e != null) {
            if (e.hash == hash && key.equals(e.key)) {
                V v = e.value;
                if (v != null)
                    return v;
                return readValueUnderLock(e); // recheck  需要的，因为new一个节点至少需要3个步骤，重拍序导致可能赋引用，再初始化对象，使得value为null。需要加锁，保证可见性。
            }
            e = e.next;
        }
    }
    return null;
}

//低Cap位得到table的下表
HashEntry<K,V> getFirst(int hash) {
    HashEntry<K,V>[] tab = table;
    return tab[hash & (tab.length - 1)];
}
```

### put操作

```java
V put(K key, int hash, V value, boolean onlyIfAbsent) {
    lock(); //对segment加锁
    try {
        int c = count;
        if (c++ > threshold) // ensure capacity
            rehash();
        HashEntry<K,V>[] tab = table;
        int index = hash & (tab.length - 1);
        HashEntry<K,V> first = tab[index];
        HashEntry<K,V> e = first;
        while (e != null && (e.hash != hash || !key.equals(e.key)))
            e = e.next;
            
        V oldValue;
        if (e != null) {
            oldValue = e.value;
            if (!onlyIfAbsent)
                e.value = value;
        }
        else {
            oldValue = null;
            ++modCount;
            tab[index] = new HashEntry<K,V>(key, hash, first, value);
            count = c; // write-volatile 保证了其他线程能看到最新的添加的Entry节点。
        }
        return oldValue;
    } finally {
        unlock();
    }
}
```

#### get时候为什么value可能会为null，需要加锁recheck
在get过程中为什么会出现v为null的情况呢。这个我之前一直想不通，自从理解了内存重拍序后才明白的。
可以看下我的另一个博客<a href="http://medesqure.top/2018/08/25/happen-before/">Happens-Before</a>的如何写一个安全的单例的例子。
其实就是内存重拍序导致的内存不可见。
比如在get时正好其他线程在执行put操作，`tab[index] = new HashEntry<K,V>(key, hash, first, value);`在new一个HashEntry时至少需要3个步骤。
1.开辟一个内存空间
2.对象初始化
3.将对象引用给栈内的局部变量。

这个过程由于指令重拍序变成1-3-2。那么就会出现value为null的情况。正好B线程执行到3，然后A线程get这个内容时得到这个引用，但是value还没有被赋值，取出的value为null。

```java
V v = e.value;
if (v != null)
    return v;
return readValueUnderLock(e);
```

### remove操作

```java
V remove(Object key, int hash, Object value) {
    lock(); //加锁
    try {
        int c = count - 1;
        HashEntry<K,V>[] tab = table;
        int index = hash & (tab.length - 1); //得到table的下标
        HashEntry<K,V> first = tab[index];
        HashEntry<K,V> e = first;
        while (e != null && (e.hash != hash || !key.equals(e.key)))
            e = e.next;


        V oldValue = null;
        if (e != null) {
            V v = e.value;
            if (value == null || value.equals(v)) {
                oldValue = v;
                // All entries following removed node can stay
                // in list, but all preceding ones need to be
                // cloned.
                //在删除节点后面的节点可以保留，但是节点之前的节点都需要重新赋值。因为引用是final类型的，不能改变
                ++modCount;
                HashEntry<K,V> newFirst = e.next; // 先新建一个newFirst，指向Node4，然后再新建一个Node1指向newFirst，然后再新建一个Node2指向Node1。
                for (HashEntry<K,V> p = first; p != e; p = p.next)
                    newFirst = new HashEntry<K,V>(p.key, p.hash,
                                                  newFirst, p.value);
                tab[index] = newFirst;
                count = c; // write-volatile
            }
        }
        return oldValue;
    } finally {
        unlock();
    }
}
```
 remove操作是先找到要删除元素的位置，不能简单的把要删除元素的前面一个元素的next指针指向后面一个元素就可以了。因为HashEntry的next是final类型，赋值后不能改变，那么就要把待删除元素的前面元素都复制一遍，在一个链接到数组上面。
![CA0E4E86-578D-45B2-8876-E3586B49ED4A](http://pbhb4py13.bkt.clouddn.com/2018-09-12-CA0E4E86-578D-45B2-8876-E3586B49ED4A.png)

### 并发的保证
在1.7的版本中，get操作是不需要加锁的，所以需要考虑get和put之间的先后顺序。

1. put操作的线程安全性 
   segement继承了reentrantlock，在put时会调用lock独占锁操作，那么保证每次只有一个线程才能put进去。而默认是16个segment，所以concurrenthashmap可以支持16个线程并发访问。
2. get和put的先后顺序
    因为table是volatile的，并且value也是volatile的，那么保证了可见性，即每次put后，get一定能获得最新的table和HashEntry节点。
    
### 1.8版本的COncurrentHashmap
在1.8版本中，**放弃使用了Segment，而是采用对table的每个槽都使用synchonorized加锁操作**，因为synchonorized使用了偏向锁和轻量级锁的优化，同时两个线程put到一个槽的概率很低，所以升级到重量级锁的概率很低。在性能上也不差很多。和18版本的HashMap结构类似，但是需要保证线程安全性。

![](http://pbhb4py13.bkt.clouddn.com/2018-09-13-15368538445516.png)

### put操作

```java
final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
    //求key的hash值，采用key的hashcode高16位和第16位异或
    int hash = spread(key.hashCode());
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        //如果table为空，初始化table
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();
        //找到hash对应的table下标记为i，第一个节点为f
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            //如果f为空，对i的槽CAS操作放入新的Node，
            //如果失败，表示其他线程放入Node，for循环重试
            if (casTabAt(tab, i, null,
                         new Node<K,V>(hash, key, value, null)))
                break;                   // no lock when adding to empty bin
        }
        //如果f的hash是MOVED，表示数组正在扩容
        else if ((fh = f.hash) == MOVED)
            //帮助扩容
            tab = helpTransfer(tab, f);
        else {
            V oldVal = null;
            //表示f节点不为空，对table的头结点f加监视器锁
            synchronized (f) {
                if (tabAt(tab, i) == f) {  //double check 
                    if (fh >= 0) { //头结点的hash大于0，表示链表
                        binCount = 1;//链表的长度计数
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            //如果key相等，即hash相等且key的值相等，那么就覆盖旧的值
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            //插入到链表的最末端
                            Node<K,V> pred = e;
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key,
                                                          value, null);
                                break;
                            }
                        }
                    }
                    //红黑树
                    else if (f instanceof TreeBin) {
                        Node<K,V> p;
                        binCount = 2;
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                       value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }
            //binCount不为0，表示为链表操作
            if (binCount != 0) {
                if (binCount >= TREEIFY_THRESHOLD)
                    //如果链表的长度大于8，会将链表转化成红黑树
                    //同时，如果还会判断数组是否需要扩容
                    treeifyBin(tab, i);
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    addCount(1L, binCount);
    return null;
}
```

其中，有几个重要的函数是数组初始化initTable(),数组的扩容或者红黑树化treeifyBin(),helpTransfer()帮助扩容操作。
### 数组初始化initTable()

```java
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    while ((tab = table) == null || tab.length == 0) {
        if ((sc = sizeCtl) < 0) //表示其他线程进行initTable操作
            Thread.yield(); // lost initialization race; just spin
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) { //先设置sizeCtl为-1
            try {
                if ((tab = table) == null || tab.length == 0) {
                    //初始化table
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    //设置sizeCtl，即Cap为0.75
                    sc = n - (n >>> 2);
                }
            } finally {
                sizeCtl = sc;
            }
            break;
        }
    }
    return tab;
}
```

#### 链表转红黑树或者数组扩容treeifyBin()
treeifyBin不一定会做红黑树转化，而且还会做数组的扩容操作。

```java
private final void treeifyBin(Node<K,V>[] tab, int index) {
    Node<K,V> b; int n, sc;
    if (tab != null) {
        //当数组的长度小于64，即32或者16时会进行数组扩容操作
        if ((n = tab.length) < MIN_TREEIFY_CAPACITY)
            tryPresize(n << 1); //扩容的方法
        else if ((b = tabAt(tab, index)) != null && b.hash >= 0) {
            //b为table的头结点，加锁访问
            synchronized (b) {
                if (tabAt(tab, index) == b) {
                    TreeNode<K,V> hd = null, tl = null;
                    for (Node<K,V> e = b; e != null; e = e.next) {
                        TreeNode<K,V> p =
                            new TreeNode<K,V>(e.hash, e.key, e.val,
                                              null, null);
                        if ((p.prev = tl) == null)
                            hd = p;
                        else
                            tl.next = p;
                        tl = p;
                    }
                    //将红黑树设置到数组的位置上
                    //treeNode继承了Node
                    setTabAt(tab, index, new TreeBin<K,V>(hd)); 
                }
            }
        }
    }
}
```

#### 数组的扩容操作 tryPresize()

```java
//size已经翻倍了
private final void tryPresize(int size) {
    // c：size 的 1.5 倍，再加 1，再往上取最近的 2 的 n 次方。
    int c = (size >= (MAXIMUM_CAPACITY >>> 1)) ? MAXIMUM_CAPACITY :
        tableSizeFor(size + (size >>> 1) + 1);
    int sc;
    while ((sc = sizeCtl) >= 0) {
        Node<K,V>[] tab = table; int n;
 
        // 这个 if 分支和之前说的初始化数组的代码基本上
        //是一样的，在这里，我们可以不用管这块代码
        //tab的初始化操作
        if (tab == null || (n = tab.length) == 0) {
            n = (sc > c) ? sc : c;
            if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                try {
                    if (table == tab) {
                        @SuppressWarnings("unchecked")
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        table = nt;
                        sc = n - (n >>> 2); // 0.75 * n
                    }
                } finally {
                    sizeCtl = sc;
                }
            }
        }
        else if (c <= sc || n >= MAXIMUM_CAPACITY)
            break;
        else if (tab == table) {
            // 我没看懂 rs 的真正含义是什么，不过也关系不大
            int rs = resizeStamp(n);
            //sc小于0，sc为什么会有小于0的情况？？？sc是个局部变量啊
            if (sc < 0) {
                Node<K,V>[] nt;
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                    transferIndex <= 0)
                    break;
                // 2. 用 CAS 将 sizeCtl 加 1，然后执行 transfer 方法
                //    此时 nextTab 不为 null
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                    transfer(tab, nt);
            }
            // 1. 将 sizeCtl 设置为 (rs << RESIZE_STAMP_SHIFT) + 2)
            //     我是没看懂这个值真正的意义是什么？不过可以计算出来的是，结果是一个比较大的负数
            //  调用 transfer 方法，此时 nextTab 参数为 null
            //这个CAS的操作是如果SIZECTL急sizeCtl的内存之=sc，
            //就把sizeCtl设置成(rs << RESIZE_STAMP_SHIFT) + 2))
            else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                         (rs << RESIZE_STAMP_SHIFT) + 2))
                transfer(tab, null); //数据迁移函数
        }
    }
}
```


