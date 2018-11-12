---
title: JDK源码之LinkedHashMap
date: 2016-12-04 13:18:58
tags: [jdk,jdk源码,LinkedHashMap]
categories: [java]
---

## LinkedHashMap使用场景
我们都知道平常使用的HashMap是存放无顺序的，但当我们需要有顺序的HashMap的时候呢？所以JDK提供了`LinkedHashMap`和`TreeMap`2种有序的Map供我们使用。虽然以前一直都看过说`LinkedHashMap`是通过链表实现的，但一直没有去源码里探索究竟，今天看了之后原来和自己理解的不完全一样，理论的这些东西，还是自己真正去研究过才有底气。
`LinkedHashMap`还支持插入顺序和访问顺序2种方式，默认的就是按照插入顺序排序的，如果需要按照访问顺序排序，在初始化时设置`accessOrder`为true即可。通过这个访问顺序我们也可以实现简单的LRU算法。

## 源码概要
`LinkedHashMap`继承了`HashMap`，所以好多方法都是直接用的`HashMap`中的，在控制底层数据这方面，并没有选择自己去实现，而是通过重写了`HashMap`中的某些节点方法，来完成相应的功能。

比如通过重写一些对节点的操作来实现了自己的链表结构，增，删，改，查，几乎都是用的`HashMap`中的方法，但是在遍历的时候，以及在需要对象顺序的地方都重写了自己有序实现，这样即保持了功能性，又避免了开发重复的功能。

<!-- more -->

## 源码分析


### 类变量
```java
    transient LinkedHashMap.Entry<K,V> head;

    transient LinkedHashMap.Entry<K,V> tail;

    final boolean accessOrder;
```
这里我们可以看到，有两个链表节点，一个头一个尾，是用来维护链表的关系的。`accessOrder`是用来控制访问顺序的。至于这里变量为什么要用`transient`来修饰**不知道为什么**，虽然知道这个变量是用来控制在序列化时，该属性不会被写入文件。

### 初始化

```java
	public LinkedHashMap(int initialCapacity,
                         float loadFactor,
                         boolean accessOrder) {
        super(initialCapacity, loadFactor);
        this.accessOrder = accessOrder;
    }
```
除了最基本的构造方法之外，`LinkedHashMap`还提供了一个可以指定顺序的构造方法，我们可以看到这里构造方法也都是调用的`HashMap`中的构造方法，`accessOrder`就是可以指定是按插入排序还是按照访问排序。


### put方法
`LinkedHashMap`并没有重写put方法，而是重写了put方法中的`newNode`方法，所以底层的数据存储和`HashMap`其实是一样的。
```java
	Node<K,V> newNode(int hash, K key, V value, Node<K,V> e) {
        LinkedHashMap.Entry<K,V> p =
            new LinkedHashMap.Entry<K,V>(hash, key, value, e);
        linkNodeLast(p);
        return p;
    }
```
在这里我们可以看到，除了正常的新建节点之外，还调用了`linkNodeLast(p)`这个方法，这里就是主要的具体实现了，就是通过这里来自己维护了一个链表。
```java
	private void linkNodeLast(LinkedHashMap.Entry<K,V> p) {
        LinkedHashMap.Entry<K,V> last = tail;
        tail = p;
        if (last == null)
            head = p;
        else {
            p.before = last;
            last.after = p;
        }
    }
```
如果是首个节点，head和tail是指向同一个节点的。
![链表初始化](http://ogflhfadi.bkt.clouddn.com/%E9%93%BE%E8%A1%A8%E5%88%9D%E5%A7%8B%E5%8C%96.png)

如果不是首节点，则进入正常的双向链表结构。
![正常结构](http://ogflhfadi.bkt.clouddn.com/%E5%8F%8C%E5%90%91%E9%93%BE%E8%A1%A8%E6%AD%A3%E5%B8%B8%E7%BB%93%E6%9E%84.png)
这里其实就是前驱和后继节点都互相指向，前驱节点也有指向后继节点的连接，后继节点也有指向前驱节点的连接。不向单向链表一样，只是前节点有后节点的连接。
每次放入一个新的节点，都会把自己设置为尾节点。所以链表的添加速度还是很快的。

#### 关于HashMap
如果大家看过我之前的关于HashMap源码的文章，会发现有几个我没弄懂的地方，没想到是在`LinkedHashMap`中使用的。
`afterNodeAccess()`和`afterNodeInsertion()`这2个方法，在`HashMap`中都是空实现，原来是为了在`LinkedHashMap`中使用。

##### afterNodeAccess
这个方法在HashMap put方法中是如果数据已经存在才会调用的。
```java
	void afterNodeAccess(Node<K,V> e) { // move node to last
        LinkedHashMap.Entry<K,V> last;
        if (accessOrder && (last = tail) != e) {
            LinkedHashMap.Entry<K,V> p =
                (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
            p.after = null;
            if (b == null)
                head = a;
            else
                b.after = a;
            if (a != null)
                a.before = b;
            else
                last = b;
            if (last == null)
                head = p;
            else {
                p.before = last;
                last.after = p;
            }
            tail = p;
            ++modCount;
        }
    }
```
这里还是比较绕的，让我们来一一的分析。
`if (accessOrder && (last = tail) != e)`
这个判断的条件是设置的是访问顺序排序，并且当前操作的这个节点是不是最后一个节点。因为在HashMap中几乎所有对节点操作的方法都调用了此方法。
```java
	LinkedHashMap.Entry<K,V> p =
                (LinkedHashMap.Entry<K,V>)e,
                b = p.before,
                a = p.after;
            p.after = null;
```
这个赋值看起来也比较恶心，这里拆开分析下。
- 首先，当前操作的节点e赋值给了p，也就是当前操作节点。
- p.before赋值给了b，也就是当前操作节点的前驱节点。
- p.after赋值给了a，也就是当前操作节点的后继节点。
- p.after给了null，证明把p设置为了尾节点。

在后面的判断条件中又分了不同的情况，在不同的判断我会用不同的图画一下。

```java
			if (b == null)
                head = a;
            else
                b.after = a;
```
这个判断的意思是

- 如果`b==null`就证明该节点是头节点，所以直接把head设置为了该节点的后继节点。
- 如果`b!=null`，就证明该节点不是头节点，所以前驱节点的后继设置为了自己节点的后继也就是`a`。这个可以说起来比较绕，意思就是，把自己从中间移除了，把前驱和后继连接了起来。

举例说明一下：

![首节点](http://ogflhfadi.bkt.clouddn.com/%E9%A6%96%E8%8A%82%E7%82%B9.png)

上图这里假设我们操作的节点是head节点，所以b是等于null的，所以代码里直接把head设置为了a,在图中也就是data1变成了首节点。

![非首节点](http://ogflhfadi.bkt.clouddn.com/%E9%9D%9E%E9%A6%96%E8%8A%82%E7%82%B9.png)
上图这里我们假设操作的节点是data1节点，所以b是不等于null的，所以把b.after也就是head的后继设置为data2。但p本身也就是data1的前驱还没有设置，我这里图没有体现出来。

```java
			if (a != null)
                a.before = b;
            else
                last = b;
```
这个判断的意思是

- 如果`a==null`，就证明该节点p是尾节点，所以把b赋值给了last，至于这里为什么这么弄，最下面判断会提及。**这里我有个问题，因为我感觉这里是防御式编程，如果a==null的情况最外层的判断应该都进不来。如果大家知道请给我留言指教**
- 如果`a!=null`就证明不是尾节点，把当前节点p的后继节点也就是a的前驱设置为自己节点的前驱。
这个和上图第2个是相辅的，上图只是设置了前驱节点的连接，这里是设置了后继节点的连接。

同样，如下图：

![尾节点](http://ogflhfadi.bkt.clouddn.com/%E5%B0%BE%E8%8A%82%E7%82%B9.png)
上图如果当前操作的是尾节点，则a就等于null，所以设置了last=b。

![非尾节点](http://ogflhfadi.bkt.clouddn.com/%E9%9D%9E%E5%B0%BE%E8%8A%82%E7%82%B9.png)
如上图，如果操作的非尾节点，这里继续假设我们操作的是data1，因为是非尾节点，所以把a.before设置为了b，设置完成后如图所示，head和data2连接上了，因为在上边的判断中head的后继已经设置过了，因此到这里基本前驱和后继的关系已经建立成功了，再往下就是把当前的操作节点p设置为尾节点。

```java
			if (last == null)
                head = p;
            else {
                p.before = last;
                last.after = p;
            }
```

- 如果`last==null`就证明还没有存放数据，所以直接设置了`head=p`，也就是首节点。
- 如果`last!=null`，就证明已经有数据存在，所以设置了当前操作节点p的前驱为last，last的后继设置为p，也就是把2个节点互相关联。last的取值有2个，一个是尾节点，也就是刚进入时候的大判断，一个是上边提到过的`a==null`的时候，所以last始终是最后一个节点。

![最后结构](http://ogflhfadi.bkt.clouddn.com/%E6%9C%80%E5%90%8E%E7%BB%93%E6%9E%84.png)
如果上图，这里继续假设我们操作的是data1，因为last取舍是tail，所以last不等于null，所以这里就把p和last互相连接最终形成了图下半部分的结构，其实就是data1换了个位置，换成最后了。

```java
tail = p;
++modCount;
```
最后，把tail设置成最后操作的节点p，并且修改次数+1。

##### afterNodeInsertion
```java
	void afterNodeInsertion(boolean evict) { // possibly remove eldest
        LinkedHashMap.Entry<K,V> first;
        if (evict && (first = head) != null && removeEldestEntry(first)) {
            K key = first.key;
            removeNode(hash(key), key, null, false, true);
        }
    }
```
这里没太看明白，为什么要删除首节点？而且这里的`removeEldestEntry`方法是返回fasle的，估计是让自己去实现，重写这个类吧。

#### newTreeNode
同node，不做过多解释。

### remove方法
```java
	void afterNodeRemoval(Node<K,V> e) { // unlink
        LinkedHashMap.Entry<K,V> p =
            (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
        p.before = p.after = null;
        if (b == null)
            head = a;
        else
            b.after = a;
        if (a == null)
            tail = b;
        else
            a.before = b;
    }
```
同样，remove方法也是重写了`afterNodeRemoval`。这里其实就是一个前驱后继的关联这里就不多说了。

### get方法
```java
	public V get(Object key) {
        Node<K,V> e;
        if ((e = getNode(hash(key), key)) == null)
            return null;
        if (accessOrder)
            afterNodeAccess(e);
        return e.value;
    }
```
get方法这里是重写了的，这里同理，先是直接调用HashMap中的查找方法，如果需要设置访问顺序排序，就进行操作，否则，直接返回数据。

### 迭代器
增，删，改，查的操作，其实和HashMap很类似，并且也都是共用了一些父类的方法。因为自己给护了一套链表，所以在迭代器中，这些东西就是自己的实现了。

示例代码：

```java
	public static void main(String[] args) {

        Map map = new LinkedHashMap();
        map.put("0", "a");
        map.put("1", "b");
        map.put("2", "c");
        Iterator it = map.entrySet().iterator();
        while (it.hasNext()){
            System.out.println(it.next());
        }
    }
```

输出结果：
`
0=a
1=b
2=c
`

`entrySet()`返回的是`LinkedEntrySet`这个类。此类是`LinkedHashMap`中的内部类。
`LinkedEntrySet `中的`iterator`方法返回的是`LinkedEntryIterator`这个内部类。
`LinkedEntryIterator `类最终继承了`LinkedHashIterator`这个类，所以最终的实现都在这里。


```java
	abstract class LinkedHashIterator {
        LinkedHashMap.Entry<K,V> next;
        LinkedHashMap.Entry<K,V> current;
        int expectedModCount;

        LinkedHashIterator() {
            next = head;
            expectedModCount = modCount;
            current = null;
        }

        public final boolean hasNext() {
            return next != null;
        }

        final LinkedHashMap.Entry<K,V> nextNode() {
            LinkedHashMap.Entry<K,V> e = next;
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            if (e == null)
                throw new NoSuchElementException();
            current = e;
            next = e.after;
            return e;
        }

        public final void remove() {
            Node<K,V> p = current;
            if (p == null)
                throw new IllegalStateException();
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            current = null;
            K key = p.key;
            removeNode(hash(key), key, null, false, false);
            expectedModCount = modCount;
        }
    }
```

这里可以看到经典的迭代器实现，我们最开始学的时候自己写迭代器就是这样的。
构造的时候把`next=head`所以判断hasNext很简单，只要判断next是否为空即可，因为在执行next方法的时候，next的引用指向也会变的。

nextNode()方法首先是判断是否安全删除的，这里是这样的如果在迭代器中，使用了Map自带的删除，迭代器就会抛出错误，所以，在迭代器中，只能使用迭代器的删除功能。然后设置current为当前节点，把next引用指向更新为下一个节点引用，返回节点。

## 总结
看完源码之后，更新了我对`LinkedHashMap`的认识，以前只是认为简单的双向链表实现的，没想到和`HashMap`有着莫大的关系。

个人认为`LinkedHashMap`虽然实现了有序，但是多维护了一层链表，所以我认为它的性能有可能会比`HashMap`是差的，但我用JDK1.8做了一下实验，插入100W条数据`LinkedHashMap`几乎是`HashMap`的一半。这里我自己就不是很能理解了，为什么多维护了数据，性能反而更快了呢？

这里我们也看到`LinkedHashMap`的优点和缺点几乎和`HashMap`是一样的，所以如果我们需要有序的`Map`，这是个很好的选择。
