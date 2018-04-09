---
layout:     post
title:      "ConcurrentHashMap详解"
subtitle:   " \"人生就像一场马拉松，跑到最后才是赢家。\""
date:       2018-04-08 10:30:00
author:     "Leon"
header-img: "img/2018-04-03-posts-bg.jpg"
catalog: true
tags:
    - 并发
---

> “Anyway, anyhow. ”

[本文参考文献：Java集合-ConcurrentHashMap工作原理和实现JDK8](https://www.jianshu.com/p/85d158455861)

### ConcurrentHashMap

``ConcurrentHashMap.class``位于``java.util.concurrent``下。相比较JDK1.7，JDK1.8中ConcurrentHashMap抛弃了分段锁技术(Segment)的实现，直接采用CAS + synchronize保证并发更新的安全性，底层的数据存储结构改成了：数组 + 链表 + 红黑树。其间包含了核心的静态内部类Node。下面用一张图来说明ConcurrentHashMap的数据结构：
<img class="shadow" src="/blog/img/ConcurrentHashMap.jpg" width="780" height="780">
数据结构采用数组 + 链表 + 红黑树的方式实现。当链表中(bucket)的节点个数超过``TREEIFY_THRESHOLD``值(默认为8)时，会转换成红黑树的数据结构存储。这样的设计优化了同一个链表冲突过大情况下的读取效率(o(n) -> o(lgn))。

#### Java8中的优化
- 将Segment抛弃掉了，直接采用Node(继承自Map.Entry)作为table元素。
- 修改时，不再采用ReentrantLock加锁，直接使用内置的``synchronized``加锁，java8的``synchronized``比之前的版本优化了很多，相比较ReentrantLock，性能并不差。
- size方法优化，增加了CounterCell内部类，用于并行计算每个bucket的元素数量。

#### 内部类和继承关系
Java8中ConcurrentHashMap增加了很多内部类来支持一些操作和优化性能。下面给出几个核心内部类和他们的继承关系图：
<img class="shadow" src="/blog/img/ConcurrentHashMap核心内部类关系.jpg" width="780" height="780">

- Node类：存放元素的key，value，hash值，next下一个链表节点的引用。用于bucket为链表时。
- TreeBin：内部属性有root，first节点，以及节点的锁状态lockState，这是一个读写锁的状态。用于存放红黑树的root节点，并用读写锁lockState控制在写操作(即将要调整树结构)之前。先让读线程完成读操作。从链表结构调整为红黑树时，table中索引下标存储的就是TreeBin。
- TreeNode：红黑树的节点，存放了父节点、左子节点、右子节点的引用以及红黑节点的标识。
- ForwardingNode：在调用transfer()期间，插入bucket头部的节点，主要用来标识在扩容时元素的移动状态，即是否在扩容时还有并发的插入节点，并保证该节点也能够移动到扩容后的表中。
- ReservationNode：占位节点，不存储任何信息，无实际用处，仅用于computeIfAbsent和compute方法中。

### 重要属性
```
public class ConcurrentHashMap<K,V> extends AbstractMap<K,V>
    implements ConcurrentMap<K,V>, Serializable {
    // table最大容量，为2的幂次方
    private static final int MAXIMUM_CAPACITY = 1 << 30;
    // 默认table初始容量大小
    private static final int DEFAULT_CAPACITY = 16;
    // 默认支持并发更新的线程数量
    private static final int DEFAULT_CONCURRENCY_LEVEL = 16;
    // table的负载因子
    private static final float LOAD_FACTOR = 0.75f;
    // 链表转换为红黑树的节点数阈值，超过这个值，链表转换为红黑树
    static final int TREEIFY_THRESHOLD = 8;
    // 在扩容期间，由红黑树转换为链表的阈值，小于这个值，resize期间红黑树就会转为链表
    static final int UNTREEIFY_THRESHOLD = 6;
    // 转为红黑树时，红黑树中节点的最小个数
    static final int MIN_TREEIFY_CAPACITY = 64;
    // 扩容时，并发转移节点(transfer方法)时，每次转移的最小节点数
    private static final int MIN_TRANSFER_STRIDE = 16;

    // 以下常量定义了特定节点类hash字段的值
    static final int MOVED     = -1; // ForwardingNode类对象的hash值
    static final int TREEBIN   = -2; // TreeBin类对象的hash值
    static final int RESERVED  = -3; // ReservationNode类对象的hash值
    static final int HASH_BITS = 0x7fffffff; // 普通Node节点的hash初始值

    // table数组
    transient volatile Node<K,V>[] table;
    // 扩容时，下一个容量大小的talbe，用于将原table元素移动到这个table中
    private transient volatile Node<K,V>[] nextTable;
    // 基础计数器
    private transient volatile long baseCount;
    // table初始容量大小以及扩容容量大小的参数，也用于标识table的状态
    // 其有几个值来代表也用来代表table的状态:
    // -1 ：标识table正在初始化
    // - N : 标识table正在进行扩容，并且有N - 1个线程一起在进行扩容
    // 正数：初始table的大小，如果值大于初始容量大小，则表示扩容后的table大小。
    private transient volatile int sizeCtl;
    // 扩容时，下一个节点转移的bucket索引下标
    private transient volatile int transferIndex;
    // 一种自旋锁，是专为防止多处理器并发而引入的一种锁，用于创建CounterCells时使用，
    // 主要用于size方法计数时，有并发线程插入而计算修改的节点数量，
    // 这个数量会与baseCount计数器汇总后得出size的结果。
    private transient volatile int cellsBusy;
    // 主要用于size方法计数时，有并发线程插入而计算修改的节点数量，
    // 这个数量会与baseCount计数器汇总后得出size的结果。
    private transient volatile CounterCell[] counterCells;
    // 其他省略
```
以上的一些属性，在初始化，扩容，链表转红黑树等方法中用到。属性众多，sizeCtl，counterCells都比较重要。
sizeCtl：即作为table初始化状态的标识，也用作扩容时的线程数标识，还用作初始和扩容后table的容量标识，用处很多。

### 核心方法源码分析
##### put方法
put方法，调用的是putVal方法。
```
public V put(K key, V value) {
    return putVal(key, value, false);
}
```
再看看putVal方法的实现：
```
//参数onlyIfAbsent表示如果为true，若put的位置已经有value，则不修改，putIfAbsent方法中传true。
final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
    int hash = spread(key.hashCode());// 计算key的hash值
    int binCount = 0;// 表示table中索引下标代表的链表或红黑树中的节点数量
    // 采用自旋方式，等待table第一次put初始化完成，或等待锁或等待扩容成功然后再插入
    for (Node<K,V>[] tab = table;;) {
        // f节点标识table中的索引节点，可能是链表的head，也可能是红黑树的head
        // n:table的长度，i:插入元素在table的索引下标，fh : head节点的hash值
        Node<K,V> f; int n, i, fh;
        if (tab == null || (n = tab.length) == 0)// 第一次插入元素，先执行初始化
            tab = initTable();
        // 定位到的索引下标节点(head)为null，表示第一次在此索引插入，
        // 不加锁直接插入在head之后，在casTabAt中采用Unsafe的CAS操作，保证线程安全
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            if (casTabAt(tab, i, null, new Node<K,V>(hash, key, value, null)))
                break;                   // no lock when adding to empty bin
        }
        // head节点为ForwadingNode类型节点，表示table正在扩容，链表或红黑树也加入到帮助扩容操作中
        else if ((fh = f.hash) == MOVED) 
            tab = helpTransfer(tab, f);
        else {// 索引下标存在元素，且为普通Node节点，给head加锁后执行插入或更新
            V oldVal = null;
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    if (fh >= 0) {// 为普通链表节点，还记得之前定义的几种常量Hash值吗？
                        binCount = 1;
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            if (e.hash == hash && ((ek = e.key) == key || (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            Node<K,V> pred = e;
                            // 插入新元素，每次插在单向链表的末尾，这点与Java7中不同（插在首部）
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key, value, null);
                                break;
                            }
                        }
                    }
                    else if (f instanceof TreeBin) {// head为树节点，按树的方式插入节点
                        Node<K,V> p;
                        binCount = 2;
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key, value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }
            // 链表节点树超过阈值8，将链表转换为红黑树结构
            if (binCount != 0) {
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    // 如果是插入新元素，则将链表或红黑树最新的节点数量加入到CounterCells中
    addCount(1L, binCount);
    return null;
}
```
初看起来，putVal方法很复杂，但笔者在代码上增加了比较详细的注释，看起来就方便的多啦，总体流程和步骤如下：

```
1、采用自旋的方式，保证首次put时，当前线程或其他并发put的线程等待table初始化完成后再次重试插入。
2、采用自旋的方式，检查当前插入的元素在table中索引下标是否正在执行扩容，如果正在扩容，则帮助进行扩容，完成后，重试插入到新的table中。
3、插入的table索引下标不为空，则对链表或红黑树的head节点加synchronized锁，再插入或更新。访问入口是Head节点，其他线程访问head，在链表或红黑树插入或修改时必须等待synchronized释放。
4、插入后，如果发现链表节点数大于等于阈值8，调用treeifyBin方法，将链表转换为红黑树结构，提高读写性能。treeifyBin方法内部也同样采用synchronized方式保证线程安全性。
5、插入元素后，会将索引代表的链表或红黑树的最新节点数量更新到baseCount或CounterCell中。
```
putVal方法用到了很多字方法，如下，我们一一来分析：
（1）spread：计算元素的hash值
（2）initTable：初始化table，在首次执行put，computeIfAbsent，computIfPresent,compute,merge方法时调用。
（3）tabAt：用于定位key在table中的索引节点(head节点)。
（4）casTabAt：采用Unsafe的compareAndSwapObject方法，用CAS的方式更新或替换节点。
（5）helpTransfer：帮忙扩容。
（6）treeifyBin：链表转红黑树，实现源码就不分析了，感兴趣的同学可以自行研究下。
（7）addCount：链表或红黑树节点最新数量添加到CounterCell中。

##### spread方法
计算key的hash值，将key的hashCode的高16位也加入到计算中，避免平凡冲突。如果仅用key的hashCode作为hash值，那么2,4之类的整形key值，只有低4位，那么很容易发生冲突。
```
static final int spread(int h) {
    return (h ^ (h >>> 16)) & HASH_BITS;
}
```

##### initTable方法
```
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    while ((tab = table) == null || tab.length == 0) {// while自旋
        // sizeCtl小于0，表示table正在被其他线程执行初始化，
        // 放弃初始化竞争，自旋等待初始化完成
        // 还记得前面介绍的sizeCtl的含义吗？
        if ((sc = sizeCtl) < 0)
            Thread.yield(); // lost initialization race; just spin
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
            try {
                if ((tab = table) == null || tab.length == 0) {
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    sc = n - (n >>> 2); //0.75n
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
初始化比较简单，步骤如下：
```
1、自旋检查table是否完成初始化。
2、若发现sizeCtl值为负数，则放弃初始化的竞争，让其他正在初始化的线程完成初始化。
3、如果没有其他线程初始化，则用Unsafe.compareAndSwapInt更新sizeCtl的值为-1，表示table开始被当前线程执行初始化，其他线程禁止执行。
4、初始化：table设置为默认容量大小（元素并未初始化，只是划定了大小），sizeCtl设为下次扩容table的size大小。
5、初始化完成。
```
整个初始化过程，用到了``sizeCtl``和``Unsafe.compareAndSwapInt``来保证初始化的线程安全性。

##### tabAt和casTabAt方法
这两个方法比较简单，都是利用Unsafe的CAS方法保证读取和替换的原子性，保证线程安全。
```
static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
    return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
}

static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i,
                                    Node<K,V> c, Node<K,V> v) {
    return U.compareAndSwapObject(tab, ((long)i << ASHIFT) + ABASE, c, v);
}
```
**疑问解答**：为什么table本身明明用了volatile修饰，不直接用table[i]的方式取节点，而非要用Unsafe.getObjectVolatile方法的CAS操作取节点。
**答**：虽然table本身是volatile类型，但仅仅是指table数组引用本身，而数组中每个元素并不是volatile类型，Unsafe.getObjectVolatile保证了每次从table中读取某个位置链表引用的时候都是从**主内存**中读取的，如果不用该方法，有可能读的是缓存中已有的该位置的旧数据。使用这个方法也就相当于给table中每个元素“加上了”volatile关键字。

##### helpTransfer方法
这是一个辅助扩容的方法。能够支持扩容时直接加到扩容中，其中真正扩容的核心方法是``transfer``。扩容前会更新sizeCtl的值，标识并发扩容的线程数。
```
final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
    Node<K,V>[] nextTab; int sc;
    if (tab != null && (f instanceof ForwardingNode) &&
        (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
        int rs = resizeStamp(tab.length);
        while (nextTab == nextTable && table == tab &&
               (sc = sizeCtl) < 0) {
            if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                sc == rs + MAX_RESIZERS || transferIndex <= 0)
                break;
            if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {
                transfer(tab, nextTab);
                break;
            }
        }
        return nextTab;
    }
    return table;
}

static final int resizeStamp(int n) {
    return Integer.numberOfLeadingZeros(n) | (1 << (RESIZE_STAMP_BITS - 1));
}
```

##### addCount方法
```
/**
 * Adds to count, and if table is too small and not already
 * resizing, initiates transfer. If already resizing, helps
 * perform transfer if work is available.  Rechecks occupancy
 * after a transfer to see if another resize is already needed
 * because resizings are lagging additions.
 *
 * @param x the count to add
 * @param check if <0, don't check resize, if <= 1 only check if uncontended
 */
private final void addCount(long x, int check) {
    // check,即链表或红黑树的节点数，<0不检查是否正在扩容， 
    // <=1仅检查是否存在竞争，没有竞争则直接返回
    CounterCell[] as; long b, s;
    // 如果首次执行addCount，并且尝试用CAS对baseCount计数失败，表示有竞争，则执行如下操作。
    // 或者非首次addCount，也执行如下的操作
    if ((as = counterCells) != null ||
        !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
        CounterCell a; long v; int m;
        boolean uncontended = true;
        if (as == null || (m = as.length - 1) < 0 ||
            (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
            !(uncontended =
              U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
            fullAddCount(x, uncontended);
            return;
        }
        if (check <= 1)
            return;
        s = sumCount();
    }
    if (check >= 0) {
        Node<K,V>[] tab, nt; int n, sc;
        while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
               (n = tab.length) < MAXIMUM_CAPACITY) {
            int rs = resizeStamp(n);
            if (sc < 0) {
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                    transferIndex <= 0)
                    break;
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                    transfer(tab, nt);
            }
            else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                         (rs << RESIZE_STAMP_SHIFT) + 2))
                transfer(tab, null);
            s = sumCount();
        }
    }
}
// sumCount方法
final long sumCount() {
    CounterCell[] as = counterCells; CounterCell a;
    long sum = baseCount;
    if (as != null) {
        for (int i = 0; i < as.length; ++i) {
            if ((a = as[i]) != null)
                sum += a.value;
        }
    }
    return sum;
}
```
addCount方法做了如下操作：
```
1、判断是否首次执行addCount，并判断是否存在竞争关系，如果CAS成功，数量就成功汇总到baseCount中，如果CAS操作失败，则表示有竞争，有其他线程并发插入，则修改的数量会被记录到CounterCell中。
2、BaseCount和CounterCell相加就表示正常无并发下的节点数量和并发插入下的节点数量，table索引下标所代表的链表或红黑树节点的数量就能达到精确计算的效果。
3、在addCount时，还会去检查sizeCtl是否为-N，以确定table是否正在扩容，如果正在扩容，则加入到扩容的操作中。
```
addCount方法所统计的数值baseCount和counterCells将会被用到size方法中，用于精确计算并发读写情况下table中元素的数量。

##### size方法
```
public int size() {
    long n = sumCount();
    return ((n < 0L) ? 0 :
            (n > (long)Integer.MAX_VALUE) ? Integer.MAX_VALUE :
            (int)n);
}
// sumCount方法
final long sumCount() {
    CounterCell[] as = counterCells; CounterCell a;
    long sum = baseCount;
    if (as != null) {
        for (int i = 0; i < as.length; ++i) {
            if ((a = as[i]) != null)
                sum += a.value;
        }
    }
    return sum;
}
```
size方法最终执行的是sumCount方法，在sumCount方法中，其实就是将baseCount的数值与CounterCell表中并发情况插入的节点数量进行汇总累加得到。这个结果把并发的情况也考虑进去了。

##### get方法
get方法步骤：
1、计算key的hash值，并定位table索引
2、若table索引下元素(head节点)为普通链表，则按链表的形式迭代遍历。
3、若table索引下元素为红黑树TreeBin节点，则按红黑树的方式查找(find方法)。
```
public V get(Object key) {
    Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
    int h = spread(key.hashCode());
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (e = tabAt(tab, (n - 1) & h)) != null) {
        if ((eh = e.hash) == h) {// 普通链表
            if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                return e.val;
        }
        // hash值小于-1,即为红黑树，还记得之前定义的TreeBin节点的hash值吗
        else if (eh < 0)
            return (p = e.find(h, key)) != null ? p.val : null;
        while ((e = e.next) != null) {// 匹配下一个链表元素
            if (e.hash == h &&
                ((ek = e.key) == key || (ek != null && key.equals(ek))))
                return e.val;
        }
    }
    return null;
}
```
红黑树的查找方法源码如下：
步骤如下：
1、检查lockState是否为写锁，如果是，则表示有并发写入线程在写入，则按正常的链表方式遍历并查找。
2、如果没有写锁，仅加读锁，然后按红黑树的方式查找(TreeBin.findTreeNode方法)。
```
final Node<K,V> find(int h, Object k) {
    if (k != null) {
        for (Node<K,V> e = first; e != null; ) {
            int s; K ek;
            if (((s = lockState) & (WAITER|WRITER)) != 0) {
            	//不管是链表还是树节点，第一个点仍然采用链表的方式遍历
                if (e.hash == h &&
                    ((ek = e.key) == k || (ek != null && k.equals(ek))))
                    return e;
                e = e.next;
            }
            else if (U.compareAndSwapInt(this, LOCKSTATE, s, s + READER)) {
                TreeNode<K,V> r, p;
                try {
                    p = ((r = root) == null ? null : r.findTreeNode(h, k, null));
                } finally {
                    Thread w;
                    if (U.getAndAddInt(this, LOCKSTATE, -READER) ==
                        (READER|WAITER) && (w = waiter) != null)
                        LockSupport.unpark(w);
                }
                return p;
            }
        }
    }
    return null;
}
```
**疑问解答**：前文不是说了，链表元素超过8个时，会被转成红黑树的结构吗？为什么在树节点遍历方法中，第一点仍然采用链表的方式遍历？
**回答**：还记得TreeBin和TreeNode节点和Node节点的继承关系吗？Node本身可以链成一个链表，而TreeBin和TreeNode也继承自Node节点，也自然继承了next属性，同样拥有链表的性质，其实真正在存储时，红黑树仍然是以链表形式存储的，只是逻辑上TreeBin和TreeNode多了支持红黑树的root，first, parent，left，right，red属性，在附加的属性上进行逻辑上的引用和关联，也就构造成了一颗树。这一点有点像LinkedHashMap，里面的节点又是在Table中，各个table中的元素又通过before和after引用进行双向链接，达到各个节点之间在逻辑上互链起来的效果。

红黑树的查找遍历如下，其实就是二叉树查找，红黑树是按hash值的大小来构造左子节点和右子节点的，比父节点hash值小放在左边，大则放在右边的：
```
final TreeNode<K,V> findTreeNode(int h, Object k, Class<?> kc) {
    if (k != null) {
        TreeNode<K,V> p = this;
        do  {
            int ph, dir; K pk; TreeNode<K,V> q;
            TreeNode<K,V> pl = p.left, pr = p.right;
            if ((ph = p.hash) > h)
                p = pl;
            else if (ph < h)
                p = pr;
            else if ((pk = p.key) == k || (pk != null && k.equals(pk)))
                return p;
            else if (pl == null)
                p = pr;
            else if (pr == null)
                p = pl;
            else if ((kc != null ||
                      (kc = comparableClassFor(k)) != null) &&
                     (dir = compareComparables(kc, k, pk)) != 0)
                p = (dir < 0) ? pl : pr;
            else if ((q = pr.findTreeNode(h, k, kc)) != null)
                return q;
            else
                p = pl;
        } while (p != null);
    }
    return null;
}

```
### 红黑树的原理和构造过程
这篇文章是我从作者那边搬运过来的。写的非常好，链接再开头。对红黑树的原理和构造过程感兴趣的童鞋可以移步至[ConcurrentHashMap与红黑树实现分析Java8](https://www.jianshu.com/p/b7dda385f83d)。