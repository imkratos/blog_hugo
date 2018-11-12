---
title: JDK1.8源码之ConcurrentHashMap
date: 2016-12-09 16:16:30
tags: [jdk,jdk源码,ConcurrentHashMap]
categories: [java]
---

### 什么是ConcurrentHashMap

以下观点都是建立在JDK1.8之上。

我们知道HashMap是非线程安全的，所以JDK给我们提供了几种线程安全的Map，`HashTable`，`Collections.SynchronizedMap`，`ConcurrentHashMap`几种线程安全的Map来使用。	
`HashTable`是每个方法都是synchronized修饰的。	
`Collections.SynchroinzedMap`则是用一个Object来当锁，每个方法里都使用`synchronized(obj)`来锁定。

这里没有理解以上的2种方法有什么区别，`HashTable`是作用在方法上的，所以锁的是this对象，后者是锁的Object，有什么区别吗？

最后一种`ConcurrentHashMap`则是使用了最新的锁技术来实现的。

### 实现原理


### 源码实现

#### 变量

```java
 	 // 最大的阀值
    private static final int MAXIMUM_CAPACITY = 1 << 30;

    // 如果不指定长度，默认的长度
    private static final int DEFAULT_CAPACITY = 16;

     // 数组最大长度
    static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;

    //
    private static final int DEFAULT_CONCURRENCY_LEVEL = 16;

    //与HashMap一样，负载因子也是0.75 当数组中有75%都有数据时就进行数组扩容
    private static final float LOAD_FACTOR = 0.75f;

    // 当数组单个位置链表长度超过此值之后，会修改为树结构
    static final int TREEIFY_THRESHOLD = 8;

    //当树结构节点小于6时，会修改为链表结构
    static final int UNTREEIFY_THRESHOLD = 6;

    /**
     * The smallest table capacity for which bins may be treeified.
     * (Otherwise the table is resized if too many nodes in a bin.)
     * The value should be at least 4 * TREEIFY_THRESHOLD to avoid
     * conflicts between resizing and treeification thresholds.
     */
    static final int MIN_TREEIFY_CAPACITY = 64;

    /**
     * Minimum number of rebinnings per transfer step. Ranges are
     * subdivided to allow multiple resizer threads.  This value
     * serves as a lower bound to avoid resizers encountering
     * excessive memory contention.  The value should be at least
     * DEFAULT_CAPACITY.
     */
    private static final int MIN_TRANSFER_STRIDE = 16;

    /**
     * The number of bits used for generation stamp in sizeCtl.
     * Must be at least 6 for 32bit arrays.
     */
    private static int RESIZE_STAMP_BITS = 16;

    /**
     * The maximum number of threads that can help resize.
     * Must fit in 32 - RESIZE_STAMP_BITS bits.
     */
    private static final int MAX_RESIZERS = (1 << (32 - RESIZE_STAMP_BITS)) - 1;

    /**
     * The bit shift for recording size stamp in sizeCtl.
     */
    private static final int RESIZE_STAMP_SHIFT = 32 - RESIZE_STAMP_BITS;

    /*
     * Encodings for Node hash fields. See above for explanation.
     */
    static final int MOVED     = -1; // hash for forwarding nodes
    static final int TREEBIN   = -2; // hash for roots of trees
    static final int RESERVED  = -3; // hash for transient reservations
    static final int HASH_BITS = 0x7fffffff; // usable bits of normal node hash
    
    // cpu数量
    static final int NCPU = Runtime.getRuntime().availableProcessors();
    
     /**
     * The array of bins. Lazily initialized upon first insertion.
     * Size is always a power of two. Accessed directly by iterators.
     */
    transient volatile Node<K,V>[] table;

    /**
     * The next table to use; non-null only while resizing.
     */
    private transient volatile Node<K,V>[] nextTable;

    /**
     * Base counter value, used mainly when there is no contention,
     * but also as a fallback during table initialization
     * races. Updated via CAS.
     */
    private transient volatile long baseCount;

    // 阀值用来控制数组是否要扩容,-1时表示正在初始化，-(1+n表示有几个线程在操作)
    private transient volatile int sizeCtl;

    /**
     * The next table index (plus one) to split while resizing.
     */
    private transient volatile int transferIndex;

    /**
     * Spinlock (locked via CAS) used when resizing and/or creating CounterCells.
     */
    private transient volatile int cellsBusy;

    /**
     * Table of counter cells. When non-null, size is a power of 2.
     */
    private transient volatile CounterCell[] counterCells;
    
    
```

#### UnSafe相关

```java

    private static final sun.misc.Unsafe U;
    private static final long SIZECTL;
    private static final long TRANSFERINDEX;
    private static final long BASECOUNT;
    private static final long CELLSBUSY;
    private static final long CELLVALUE;
    private static final long ABASE;
    private static final int ASHIFT;

    static {
        try {
            U = sun.misc.Unsafe.getUnsafe();
            Class<?> k = ConcurrentHashMap.class;
            //获取sizeCtl变量在内存中的偏移量
            SIZECTL = U.objectFieldOffset
                (k.getDeclaredField("sizeCtl"));
            TRANSFERINDEX = U.objectFieldOffset
                (k.getDeclaredField("transferIndex"));
            BASECOUNT = U.objectFieldOffset
                (k.getDeclaredField("baseCount"));
            CELLSBUSY = U.objectFieldOffset
                (k.getDeclaredField("cellsBusy"));
            Class<?> ck = CounterCell.class;
            CELLVALUE = U.objectFieldOffset
                (ck.getDeclaredField("value"));
            Class<?> ak = Node[].class;
            ABASE = U.arrayBaseOffset(ak);
            int scale = U.arrayIndexScale(ak);
            if ((scale & (scale - 1)) != 0)
                throw new Error("data type scale not a power of two");
            ASHIFT = 31 - Integer.numberOfLeadingZeros(scale);
        } catch (Exception e) {
            throw new Error(e);
        }
    }
```

#### 构造方法

```java
    public ConcurrentHashMap(int initialCapacity) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException();
        /** 这里直接判断了如果指定的值大于了最大长度的1/2，就直接等于最大值了
         *  这里的 MAXIMUM_CAPACITY >>> 1 等于 MAXIMUM_CAPACITY / 2
         *  initialCapacity + (initialCapacity >>> 1) + 1
         *  这里我没明白为什么要做这些操作，可能是为了求出在做散列时
         *  更合适的值
         */  
        int cap = ((initialCapacity >= (MAXIMUM_CAPACITY >>> 1)) ?
                   MAXIMUM_CAPACITY :
                   tableSizeFor(initialCapacity + (initialCapacity >>> 1) + 1));
        this.sizeCtl = cap;
    }
    
```

构造方法都很类似，这里我只举例一个。

#### put方法

```java
	final V putVal(K key, V value, boolean onlyIfAbsent) {
        if (key == null || value == null) throw new NullPointerException();
        //根据key的hashcode再次求hash
        int hash = spread(key.hashCode());
        int binCount = 0;
        //开始操作数组 这里是个死循环 会在插入数据成功后break掉
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh;
            //数组并没有在构造方法里初始化，所以在第一次put的时候，会初始化数组
            if (tab == null || (n = tab.length) == 0)
            //这里我理解的是这样的，因为外层是个死循环，所以第一次执行到这的时候会进行初始化
            //然后就进入了下一次循环，因为已经初始化完毕了，所以就会进入别的分支判断
                tab = initTable();
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            /** 进入这个判断的条件是在数组下标位置上没有节点，就证明是个新节点可以直接插入
             * 所以就进入了这层，在这层插入的时候，因为只用了一个if判断是用的cas操作
             * 所以有可能会失败，当失败时，还是因为外层是个死循环，所以会一直执行这个插入
             * 直到插入成功，break掉
             * 并且f 和i 也顺便在这里进行了赋值 i等于数组长度-1 与上 hash 
             * f 等于 数组 i 位置上的元素 
             * 
             * /
                if (casTabAt(tab, i, null,
                             new Node<K,V>(hash, key, value, null)))
                    break;                   // no lock when adding to empty bin
            }
            else if ((fh = f.hash) == MOVED)
                tab = helpTransfer(tab, f);
            else {
            /**
             * 到这个判断，就证明数组下标位置现在是有节点的，所以会有2种操作，链表和树的操作
             * 首先在这里锁住了首节点
             */
                V oldVal = null;
                synchronized (f) {
                //由与上面f 和 i 已经赋值过了，所以这里再次进行了确认是否是同一个对象
                    if (tabAt(tab, i) == f) {
                        if (fh >= 0) {
                            binCount = 1;
                            // 以下这个循环是对链表进行循环
                            for (Node<K,V> e = f;; ++binCount) {
                                K ek;
                                // 如果hash值 和key 完全相等，就证明是同一个对象
                                // 如果没有设置替换新值到这里就结束了
                                // 如果设置了就替换新值，并且结束
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {
                                    oldVal = e.val;
                                    if (!onlyIfAbsent)
                                        e.val = value;
                                    break;
                                }
                                //这里是循环完整个链表都没有发现有相等的key
                                //直接在链表最后插入新的节点，并且结束
                                Node<K,V> pred = e;
                                if ((e = e.next) == null) {
                                    pred.next = new Node<K,V>(hash, key,
                                                              value, null);
                                    break;
                                }
                            }
                        }
                        else if (f instanceof TreeBin) {
                            Node<K,V> p;
                            binCount = 2;
                            //这里判断，如果节点是树类型的，则把节点放入到树结构中
                            //同链表一样，如果设置了需要覆盖新值，更新一下
                            if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                           value)) != null) {
                                oldVal = p.val;
                                if (!onlyIfAbsent)
                                    p.val = value;
                            }
                        }
                    }
                }
                //binCount 在树中是写死的2 在链表中会随着操作的增加而增加
                if (binCount != 0) {
                //如果超过了设置的值，就需要把链表转换成树结构
                    if (binCount >= TREEIFY_THRESHOLD)
                    //转树操作，但里面还会有扩容
                        treeifyBin(tab, i);
                    // 因为上边的操作存储了oldVal，如果不为空直接返回此值，结束
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

put方法总结：
>存放对象的时候，第一次操作，会先初始化底层的数组对象，然后再进行后续的操作
>在存放元素的时候有这么几种情况：	
>1.如果计算出的hash值与上数组长度得出的下标位置为null，则可以直接插入	
>2.如果当前元素的hash值为MOVED，就证明有线程在进行扩容，就帮助一起扩容	
>3.都没有的情况下，证明当前下标存在其他元素，则进入元素追加，如果是链表就进行链表的操作，如果是树，就进行树的操作，最后添加元素完成。	
>4.添加完元素，如果需要进行树的转换，进行转换。	
>5.增加计数器，完成所有操作。 



treeifyBin方法

```java
	private final void treeifyBin(Node<K,V>[] tab, int index) {
        Node<K,V> b; int n, sc;
        if (tab != null) {
        //如果当前数组长度小于64，会直接扩容2倍，而不把当前节点轩换为树。
            if ((n = tab.length) < MIN_TREEIFY_CAPACITY)
                tryPresize(n << 1);
                //获取index位置的节点，并且判断hash值为正常的节点，因为有可能存在-1等情况
            else if ((b = tabAt(tab, index)) != null && b.hash >= 0) {
            //锁住首节点
                synchronized (b) {
                    if (tabAt(tab, index) == b) {
                        TreeNode<K,V> hd = null, tl = null;
                        //遍历所有节点，转换为树节点
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
                        //然后把TreeNode包装成TreeBin做为数组下标的对象
                        setTabAt(tab, index, new TreeBin<K,V>(hd));
                    }
                }
            }
        }
    }
```

treeifyBin方法总结：
>如果当前数组长度小于64，则不进行树元素的转换，直接扩容成2倍，如果不是的话，那就对节点进行转换，从链表节点，转换为树节点。

tryPresize方法

```java
	private final void tryPresize(int size) {
	//根据给出的长度，计算出实际的长度
        int c = (size >= (MAXIMUM_CAPACITY >>> 1)) ? MAXIMUM_CAPACITY :
            tableSizeFor(size + (size >>> 1) + 1);
        int sc;
        //sc>=0的条件下才会进入SizeCtl的具体值见定义
        while ((sc = sizeCtl) >= 0) {
            Node<K,V>[] tab = table; int n;
            //数组没有初始化的情况，也就是一次都没有初始化，我个人感觉不会执行到这呢？
            if (tab == null || (n = tab.length) == 0) {
                n = (sc > c) ? sc : c;
                //把sizeCtl值修改为-1，为正在处理中
                if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                    try {
                    //这里再次判断了2个对象是否是一个对象，估计是为了防止其他对象操作？
                        if (table == tab) {
                        //初始化数组
                            @SuppressWarnings("unchecked")
                            Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                            table = nt;
                            sc = n - (n >>> 2);
                        }
                    } finally {
                    //最后修改sizeCtl的值，这里为什么没有用CAS操作，我理解是因为前边的if里已经修改为了-1，其他的线程不能操作，所以这里只用了普通的赋值
                        sizeCtl = sc;
                    }
                }
            }
            //如果没到0.75数，或者已经大于最大值，停止此方法
            else if (c <= sc || n >= MAXIMUM_CAPACITY)
                break;
                //到这里就证明数组是已经初始化过了的，并且需要扩容
            else if (tab == table) {
                int rs = resizeStamp(n);
                //这里是判断sc的值小于0证明在处理
                if (sc < 0) {
                    Node<K,V>[] nt;
                    //没看懂什么意思
                    if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                        sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                        transferIndex <= 0)
                        break;
                        //把sizeCtl值+1并且进行扩容
                    if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                        transfer(tab, nt);
                }
                //(rs << RESIZE_STAMP_SHIFT) + 2)没看懂求出来的这个值是什么，sc的值替换成这个之后，也进行扩容
                else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                             (rs << RESIZE_STAMP_SHIFT) + 2))
                    transfer(tab, null);
            }
        }
    }
```

tryPresize方法总结：
>这里就是对数组的一些异常情况做了兼容处理，最终保证扩容完成。

transfer方法

```java
	private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
        int n = tab.length, stride;
        //求出处理数组的线程数，最大是16个线程
        if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
            stride = MIN_TRANSFER_STRIDE; // subdivide range
        if (nextTab == null) {            // initiating
        //构造nextTab，长度为原来的2倍
            try {
                @SuppressWarnings("unchecked")
                Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
                nextTab = nt;
            } catch (Throwable ex) {      // try to cope with OOME
            //如果失败，则设置sc的最大值为int的最大值
                sizeCtl = Integer.MAX_VALUE;
                return;
            }
            nextTable = nextTab;
            //这里我理解的就是，因为是原来的2倍，所以转换的下标是n
            transferIndex = n;
        }
        int nextn = nextTab.length;
        //构造节点元素，用来标明此节点是正在处理的节点，此节点的hash值为-1
        ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
        boolean advance = true;
        boolean finishing = false; // to ensure sweep before committing nextTab
        for (int i = 0, bound = 0;;) {
            Node<K,V> f; int fh;
            while (advance) {
                int nextIndex, nextBound;
                if (--i >= bound || finishing)
                    advance = false;
                else if ((nextIndex = transferIndex) <= 0) {
                    i = -1;
                    advance = false;
                }
                else if (U.compareAndSwapInt
                         (this, TRANSFERINDEX, nextIndex,
                          nextBound = (nextIndex > stride ?
                                       nextIndex - stride : 0))) {
                    bound = nextBound;
                    i = nextIndex - 1;
                    advance = false;
                }
            }
            if (i < 0 || i >= n || i + n >= nextn) {
                int sc;
                if (finishing) {
                    nextTable = null;
                    table = nextTab;
                    sizeCtl = (n << 1) - (n >>> 1);
                    return;
                }
                if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                    if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                        return;
                    finishing = advance = true;
                    i = n; // recheck before commit
                }
            }
            else if ((f = tabAt(tab, i)) == null)
                advance = casTabAt(tab, i, null, fwd);
            else if ((fh = f.hash) == MOVED)
                advance = true; // already processed
            else {
            //锁住当前元素节点
                synchronized (f) {
                //再次确认节点是否相等，防止其他线程修改此节点
                    if (tabAt(tab, i) == f) {
                        Node<K,V> ln, hn;
                        //链表
                        if (fh >= 0) {
                        //这里我不太理解为什么是&n，因为在普通put值的时候是&(n-1)，这里求出的其实也是个下标位置
                            int runBit = fh & n;
                            Node<K,V> lastRun = f;
                            for (Node<K,V> p = f.next; p != null; p = p.next) {
                            //这里对链表后面的元素也都进行了求值，但我个人理解这里的b应该绝对等于runBit才对啊，因为他们在put的时候被放入了同一个桶，就应该是hash值相同才对的
                                int b = p.hash & n;
                                if (b != runBit) {
                                    runBit = b;
                                    lastRun = p;
                                }
                            }
                            if (runBit == 0) {
                                ln = lastRun;
                                hn = null;
                            }
                            else {
                                hn = lastRun;
                                ln = null;
                            }
                            for (Node<K,V> p = f; p != lastRun; p = p.next) {
                                int ph = p.hash; K pk = p.key; V pv = p.val;
                                if ((ph & n) == 0)
                                    ln = new Node<K,V>(ph, pk, pv, ln);
                                else
                                    hn = new Node<K,V>(ph, pk, pv, hn);
                            }
                            setTabAt(nextTab, i, ln);
                            setTabAt(nextTab, i + n, hn);
                            setTabAt(tab, i, fwd);
                            advance = true;
                        }
                        else if (f instanceof TreeBin) {
                            TreeBin<K,V> t = (TreeBin<K,V>)f;
                            TreeNode<K,V> lo = null, loTail = null;
                            TreeNode<K,V> hi = null, hiTail = null;
                            int lc = 0, hc = 0;
                            for (Node<K,V> e = t.first; e != null; e = e.next) {
                                int h = e.hash;
                                TreeNode<K,V> p = new TreeNode<K,V>
                                    (h, e.key, e.val, null, null);
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
                            ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                                (hc != 0) ? new TreeBin<K,V>(lo) : t;
                            hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                                (lc != 0) ? new TreeBin<K,V>(hi) : t;
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

transfer方法总结：
>

addCount方法

```java
	private final void addCount(long x, int check) {
        CounterCell[] as; long b, s;
        //使as等于counterCells 并且不等于null 
        //或者在修改baseCount时失败，才会进入此方法
        //这里的x值，我个人理解的也就是此次增加了几个元素。
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
```

#### remove方法


#### get方法



#### 其他相关


### 总结