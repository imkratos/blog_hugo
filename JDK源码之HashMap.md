---
title: JDK源码之HashMap
date: 2016-11-12 15:39:10
tags: [java,jdk源码,hashMap,map]
categories: [java]
---

#### 什么是HashMap
HashMap是一个可以提供O(1)时间复杂度的数据结构，由数组和链表数据结构组成。在对存入的key进行hash之后，然后用hash值在数组上确定一个位置，把value对象以Node节点形式放入到数组的链表当中。

jdk1.8之后对此做了优化，因为如果发生了数据倾斜，可能会使数组某个下标的Node链表非常长，因为链表查询起来比较慢，所以1.8之后修改了，当Node链表长度大于8时，会把该下标位置的链表数据结构修改为红黑树的结构来保证查询的速度。当数据长度小于8时，会再修改为链表。

#### 使用场景
个人理解使用场景应该是在不需要复杂的查询，只需要一个Key对应一个Value，写入少的场景。因为像HashMap，ArrayList这种数据结构都提供了自动扩容的功能，像HashMap的负载因子是0.75，也就是当数组中75%的位置都有值以后会进行扩容。每次扩容的时候都涉及到每个数据的rehash和数组的复制，所以当写入数据量非常大的时候，会不断的进行rehash和复制，有可能会造成CPU占用率非常高(这只是个人平时学习的理解，如果有不对之处请大家指正)。

所以个人感觉HashMap的使用场景也是读多写少的场景，可以提供很快速度的读，写入的速度也可以，但如果提前知道Map里要放入多少数据，最好在new对象的时候，就手动指定出长度，这样可以避免rehash，从来变相提高使用效率。


#### 源码实现 jdk1.8版本
**以下的代码都是代码在上面，解释在下面。**

##### 变量声明

```java
	 /**
     * The default initial capacity - MUST be a power of two.
     */
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16
```

这个值是HashMap默认初始化的长度，也就是16长。

```java
	/**
     * The maximum capacity, used if a higher value is implicitly specified
     * by either of the constructors with arguments.
     * MUST be a power of two <= 1<<30.
     */
    static final int MAXIMUM_CAPACITY = 1 << 30; //1073741824
```

这里按注释看，应该是HashMap支持的最大长度，如果超过这个值，则使用这个值。

<!--more-->

```java
	/**
     * The load factor used when none specified in constructor.
     */
    static final float DEFAULT_LOAD_FACTOR = 0.75f;
```
负载因子，当数组中有值的位数超过此阀值时，进行扩容。

```java
	/**
     * The bin count threshold for using a tree rather than list for a
     * bin.  Bins are converted to trees when adding an element to a
     * bin with at least this many nodes. The value must be greater
     * than 2 and should be at least 8 to mesh with assumptions in
     * tree removal about conversion back to plain bins upon
     * shrinkage.
     */
    static final int TREEIFY_THRESHOLD = 8;
```
当数组单个位置超过此值后，会把数据结构修改为红黑树。

```java
	/**
     * The bin count threshold for untreeifying a (split) bin during a
     * resize operation. Should be less than TREEIFY_THRESHOLD, and at
     * most 6 to mesh with shrinkage detection under removal.
     */
    static final int UNTREEIFY_THRESHOLD = 6;
```

当小于这个值时，修改为链表。

```java
	/**
     * The smallest table capacity for which bins may be treeified.
     * (Otherwise the table is resized if too many nodes in a bin.)
     * Should be at least 4 * TREEIFY_THRESHOLD to avoid conflicts
     * between resizing and treeification thresholds.
     */
    static final int MIN_TREEIFY_CAPACITY = 64;
```

???

```java
	transient Node<K,V>[] table;
```
存放元素的数组。

```java
	/**
     * The next size value at which to resize (capacity * load factor).
     *
     * @serial
     */
    // (The javadoc description is true upon serialization.
    // Additionally, if the table array has not been allocated, this
    // field holds the initial array capacity, or zero signifying
    // DEFAULT_INITIAL_CAPACITY.)
    int threshold;
```
这个变量是用来存放阀值的，也就是数组长度*0.75的值。

##### 初始化
```java

	/**
     * Constructs an empty <tt>HashMap</tt> with the default initial capacity
     * (16) and the default load factor (0.75).
     */
    public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
    }
	/**
     * Constructs an empty <tt>HashMap</tt> with the specified initial
     * capacity and the default load factor (0.75).
     *
     * @param  initialCapacity the initial capacity.
     * @throws IllegalArgumentException if the initial capacity is negative.
     */
    public HashMap(int initialCapacity) {
        this(initialCapacity, DEFAULT_LOAD_FACTOR);
    }
    
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
        this.threshold = tableSizeFor(initialCapacity);
    }
```

* 初始化时，如果是空的构造方法，会只设置负载因子的值为默认的0.75.
* 如果指定的`initialCapacity`超过`MAXIMUM_CAPACITY `，则值就为`MAXIMUM_CAPACITY `
* tableSizeFor函数会求出一个值作为HashMap的容量
 
```java
	/**
     * Returns a power of two size for the given target capacity.
     */
    static final int tableSizeFor(int cap) {
        int n = cap - 1;
        n |= n >>> 1;
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }
```
* 初始化完毕

##### put方法
```java
	public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }

	final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        else {
            Node<K,V> e; K k;
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
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
            resize();
        afterNodeInsertion(evict);
        return null;
    }
```

说下参数的意思，`hash`就是通过key计算出的hash值，`Key Value`不用多说就是我们要放入的Key和Value，`onlyIfAbsent`这里我理解应该是，如果Key存在是否要替换对应Key的Value，这里传入的是`false`，`evict`应该是，是否自动扩容，默认是`true`。

**声明变量**	
`tab`数组，就是需要存放节点的数组。	
`p`则是放在数组下标位置的节点。	
`n,i`数组长度和位置下标。

**代码逻辑**	
```java
	if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
```
如果数组没有初始化或者长度为0，则进行初始化。	
`resize()`方法中的初始化代码

##### 扩容机制
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
        Node<K,V>[] oldTab = table;
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        int newCap, newThr = 0;
        if (oldCap > 0) {
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                newThr = oldThr << 1; // double threshold
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
            Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;
        if (oldTab != null) {
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null;
                    if (e.next == null)
                        newTab[e.hash & (newCap - 1)] = e;
                    else if (e instanceof TreeNode)
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    else { // preserve order
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
                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
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

```java
	if (oldCap > 0) {
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                newThr = oldThr << 1; // double threshold
    }
```
1.  这里判断现有的数组长度是否为0，如果当前数组长度大于0，并且大于或等于`MAXIMUM_CAPACITY`，则`threshold`等于int的最大值，返回当前数组。

2. `newCap`的值等于当前数组的长度\*2`old<<1`，如果这个新值小于`MAXIMUM_CAPACITY `并且当前数组的长度 >= `DEFAULT_INITIAL_CAPACITY`也就是16，`newThr = oldThr << 1;`这句话的意思是新的阀值=老阀值\*2。其实以上的意思就是如果符合条件了，把新数组的长度和阀值都扩大2倍。

```java
	else if (oldThr > 0) // initial capacity was placed in threshold
            newCap = oldThr;
```
这里的意思是，如果老数组长度不大于0，并且老阀值大于0，把老阀值的值赋给新的数组长度，**这里我没理解为什么要这么做**。

```java
	else {               // zero initial threshold signifies using defaults
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
```
如果以上条件都不满足，就会进行默认的初始化。也就是数组长度等于16，阀值等于12。

```java
	if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
```
至此如果新的阀值还是0的话，会再次进行初始化，公式还是一样的，先根据新的数组长度*负载因子计算出阀值，如果新的数据长度小于长度最大值，并且三目表达式用来判断是否过界，如果不过界，返回计算的值，如果过界，返回int最大值。
```java
threshold = newThr;
Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
table = newTab;
```
到这里，基本上阀值和新的数组长度都已经初始化完了，创建了新的数组，开始初始化对象了。如果是构造方法里调用这里，到这里就已经结束了，因为数组，阀值，都已经初始化完毕了。
再下面的代码是扩容的时候会进行调用的。

```java
	if (oldTab != null) {
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null;
                    if (e.next == null)
                        newTab[e.hash & (newCap - 1)] = e;
                    else if (e instanceof TreeNode)
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    else { // preserve order
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
                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                        if (hiTail != null) {
                            hiTail.next = null;
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
```
- 到第7行为止，是表示如果老数组的下标位置只有一个节点没有链表也没有红黑树，就会把该位置的数组赋值给新数组，在新数组的位置是hash值&数组的长度(这里我以前一直认为是取余)。
- 如果是红黑树，则进行树拆分`((TreeNode<K,V>)e).split(this, newTab, j, oldCap);`

##### 红黑树
```java
	final void split(HashMap<K,V> map, Node<K,V>[] tab, int index, int bit) {
            TreeNode<K,V> b = this;
            // Relink into lo and hi lists, preserving order
            TreeNode<K,V> loHead = null, loTail = null;
            TreeNode<K,V> hiHead = null, hiTail = null;
            int lc = 0, hc = 0;
            for (TreeNode<K,V> e = b, next; e != null; e = next) {
                next = (TreeNode<K,V>)e.next;
                e.next = null;
                if ((e.hash & bit) == 0) {
                    if ((e.prev = loTail) == null)
                        loHead = e;
                    else
                        loTail.next = e;
                    loTail = e;
                    ++lc;
                }
                else {
                    if ((e.prev = hiTail) == null)
                        hiHead = e;
                    else
                        hiTail.next = e;
                    hiTail = e;
                    ++hc;
                }
            }

            if (loHead != null) {
                if (lc <= UNTREEIFY_THRESHOLD)
                    tab[index] = loHead.untreeify(map);
                else {
                    tab[index] = loHead;
                    if (hiHead != null) // (else is already treeified)
                        loHead.treeify(tab);
                }
            }
            if (hiHead != null) {
                if (hc <= UNTREEIFY_THRESHOLD)
                    tab[index + bit] = hiHead.untreeify(map);
                else {
                    tab[index + bit] = hiHead;
                    if (loHead != null)
                        hiHead.treeify(tab);
                }
            }
        }
```

这里我大概理解是这样的，先对所有的节点进行遍历，然后通过一定条件来判断是否小于等于`UNTREEIFY_THRESHOLD`这个阀值，如果小于就切换为普通的链表，如果大于就继续在树上添加节点，由于博主对树现在理解不是很好，这里暂且这样，以后我会指定研究一下树的节构。

```java
	else { // preserve order
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
                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                        if (hiTail != null) {
                            hiTail.next = null;
                            newTab[j + oldCap] = hiHead;
                        }
                    }
```

这里就是链表节构了，也是把链表循环先读取出来。但是这里为什么分成了2个链表？并且分别放在了不同的位置上。至此，resize()方法执行完毕，让我们再回来继续看put方法。
- 以上我们写了resize()中详细的方法
- 有初始化数组的长度和阀值
- 对数组进行扩容时对链表，单节点，以及树不同的处理


```java
	if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
```
这里是数组长度对hash值求与值，如果该位置为null证明该位置没有放入元素，则放入新的元素。

```java
	Node<K,V> e; K k;
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
```
这里判断，如果hash值相等并且key值相等，或者key的equest相等，就直接替换该位置的值。

```java
	else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
```
如果该位置的结构是树的，则调用树的插入方法，关于树的方法博主在这里暂不讨论，因为博主也不是特别理解树结构，以后会补充。

```java
	for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
```
如果是链表的话，循环所有的链表，如果有相等的key，直接替换该key对应的值，如果循环到最后也没有相等的，在链表上插入一个新的node节点，同时判断是否满足到达`TREEIFY_THRESHOLD`的条件，如果到达，会把数据结构变更为红黑树。

```java
	if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
```
如果到这里e已经存在了，因为e等于新放入的node节点。就把该节点移到链表最后一个，并且直接返回value值。

```java
	++modCount;
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
```
记录修改的次数`modCount`是用来控制非法修改hashmap里的值，来抛出`ConcurrentModificationException`。并且判断数组长度是否大于阀值，如果大于了就要调用`resize()`进行扩容，`afterNodeInsertion`没有理解作用是什么，至此put方法全部执行完毕。

##### get方法
```java
	final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {
            if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))
                return first;
            if ((e = first.next) != null) {
                if (first instanceof TreeNode)
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
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
put方法核心的代码是这里，这里是判断如果在数据里找到当前key的位置之后，首先判断hash值，和key是否都相等，如果都相等，直接返回此对象，如果不相等还分2种情况一种是树结构，会调用树的查找方法，另外一种是链表，则会对链表不断的循环判断，直到找到相等元素为止。
如果最后都没有找到，则返回null。

##### remove方法
```java
	final Node<K,V> removeNode(int hash, Object key, Object value,
                               boolean matchValue, boolean movable) {
        Node<K,V>[] tab; Node<K,V> p; int n, index;
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (p = tab[index = (n - 1) & hash]) != null) {
            Node<K,V> node = null, e; K k; V v;
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                node = p;
            else if ((e = p.next) != null) {
                if (p instanceof TreeNode)
                    node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
                else {
                    do {
                        if (e.hash == hash &&
                            ((k = e.key) == key ||
                             (key != null && key.equals(k)))) {
                            node = e;
                            break;
                        }
                        p = e;
                    } while ((e = e.next) != null);
                }
            }
            if (node != null && (!matchValue || (v = node.value) == value ||
                                 (value != null && value.equals(v)))) {
                if (node instanceof TreeNode)
                    ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
                else if (node == p)
                    tab[index] = node.next;
                else
                    p.next = node.next;
                ++modCount;
                --size;
                afterNodeRemoval(node);
                return node;
            }
        }
        return null;
    }
```
remove方法和get方法类似，核心代码是首先也是在数组里找位置，如果当前位置首个节点是这个key，就把值给node，如果当前值不是，则进行后续查找。如果是树，调用树的查找方法，把查找到的值给node，如果是链表，则对链表进行循环查找，找到之后把值给node。
最后判断如果node不这代，并且value值相等。则会进行删除动作，如果是树节构就调用树的删除方法，如果数组的位置首节点是要删除的值，则直接把next的值给当前位置，如果是链表后续的值，则把要删除的key的下一个节点，和上一个节点连接，自己就会被删除掉。
最后增加修改次数，size减掉。返回删除的节点。


