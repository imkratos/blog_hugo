---
title: JDK源码之CopyOnWriteArrayList
date: 2016-11-10 23:14:52
tags: [java,jdk,jdk源码,CopyOnWriteArrayList]
categories: [java]
---

#### 什么是CopyOnWriteArrayList

CopyOnWriteArrayList是一个线程安全的List，和它相同的还有这几个,Vectory,SynchronizedList。
它们都是实现了List接口的线程安全类。

CopyOnWriteArrayList使用的是一种写时复制的算法，也就是说在执行add方法的时候，并不像传统的ArrayList一样在当前数组直接操作，
而是在执行add方法的时候，会把原来的数组复制并且长度+1，然后把新值设置到新的数组中，然后把新数组设置为当前使用状态，由于数组是`volatile`修饰的，所以JVM会自动来保证数组的可见性。

这样使此List在读取时可以不用加锁，提高读取效率，但是在添加的时候效率会很低下。所以我感觉CopyOnWriteArrayList的使用场景就是读多写少的场景，甚至在尽可能的情况下，不去写。这样才能发挥此数据结构的最大优点。
<!--more-->

#### 源码实现


```java
    /** The lock protecting all mutators */
    final transient ReentrantLock lock = new ReentrantLock();
    /** The array, accessed only via getArray/setArray. */
    private transient volatile Object[] array;
```
数组结构和可重入锁，由于数组是`volatile`修饰的，所以JVM会利用CPU的缓存失效功能，将对象保持强一致性。假设有4个CPU，因为每个CPU都有属于自己的高速缓存，在此类对象进行更改时，JVM会调用CPU方法，将所有CPU的缓存失效，并且把修改后的内容刷回主存来保持此对象的强一致性，不知道这里我理解的有没有问题，如果有问题请各位指正。
```java
    /**
     * Creates an empty list.
     */
    public CopyOnWriteArrayList() {
        setArray(new Object[0]);
    }
```
默认初始化，是一个0长度的数组。

```java
  /**
     * Appends the specified element to the end of this list.
     *
     * @param e element to be appended to this list
     * @return {@code true} (as specified by {@link Collection#add})
     */
    public boolean add(E e) {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            Object[] elements = getArray();
            int len = elements.length;
            Object[] newElements = Arrays.copyOf(elements, len + 1);
            newElements[len] = e;
            setArray(newElements);
            return true;
        } finally {
            lock.unlock();
        }
    }
```
这里便是添加元素的代码，这里使用了重入锁，重入锁的意思是如果当前线程已经拿到了锁，再次获取此对象上的锁的时候，也会获得到锁不会排队。

上锁以后，先得到当前的数组，并取得当前的长度，然后调用`Arrays.copyOf`方法，复制数组并且长度+1，然后把新的元素加到新的数组中，设置新的数组为当前使用的数组。返回添加成功，并且释放锁。



```java
    /**
     * Removes the element at the specified position in this list.
     * Shifts any subsequent elements to the left (subtracts one from their
     * indices).  Returns the element that was removed from the list.
     *
     * @throws IndexOutOfBoundsException {@inheritDoc}
     */
    public E remove(int index) {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            Object[] elements = getArray();
            int len = elements.length;
            E oldValue = get(elements, index);
            int numMoved = len - index - 1;
            if (numMoved == 0)
                setArray(Arrays.copyOf(elements, len - 1));
            else {
                Object[] newElements = new Object[len - 1];
                System.arraycopy(elements, 0, newElements, 0, index);
                System.arraycopy(elements, index + 1, newElements, index,
                                 numMoved);
                setArray(newElements);
            }
            return oldValue;
        } finally {
            lock.unlock();
        }
    }
```
删除这里基本同增加一样，也是先加锁，然后再操作。

这里不同的是使用了`int numMoved = len - index - 1;`这个值来判断删除的元素是不是最后一个元素，如果是最后一个元素也就是`numMoved=0`，直接把原来的数组复制长度-1即可。

如果不是最后一个元素，则是分开复制的，先复制删除元素前面的数据，再复制删除元素后面的数据，最后形成新的数组，设置并返回，最后解锁。




```java
    /**
     * Replaces the element at the specified position in this list with the
     * specified element.
     *
     * @throws IndexOutOfBoundsException {@inheritDoc}
     */
    public E set(int index, E element) {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            Object[] elements = getArray();
            E oldValue = get(elements, index);

            if (oldValue != element) {
                int len = elements.length;
                Object[] newElements = Arrays.copyOf(elements, len);
                newElements[index] = element;
                setArray(newElements);
            } else {
                // Not quite a no-op; ensures volatile write semantics
                setArray(elements);
            }
            return oldValue;
        } finally {
            lock.unlock();
        }
    }
```
修改操作也类似，先加锁，得到原值，如果原值和修改的元素相等，其实感觉什么也没干，看这里注释只是为了保持volatile的语法。如果不等的话拷贝一个新的数组，在新的数组上修改值，把新的数组设置为当前使用的数组，返回修改之前的元素，解锁，完成。


```java
    /**
     * {@inheritDoc}
     *
     * @throws IndexOutOfBoundsException {@inheritDoc}
     */
    public E get(int index) {
        return get(getArray(), index);
    }
```
下标查找，这里看到和普通的数组没什么区别，就是普通的下标查找。



#### 总结
从源码来看，CopyOnWriteArrayList增，删，改，都会复制一下新的数组然后再设置回去，所以强烈建议此数据结构的使用场景基本是只读的，否则在大量操作的情况下性能应该会很慢(待以后数据验证)。
