---
title: JDK源码之ThreadLocal
date: 2016-11-26 21:07:05
tags: [jdk,jdk源码]
categories: [java]
---
#### ThreadLocal介绍
一般我们都是如果发现有资源需要共享的时候，在多个线程之间要互相共享数据的时候，我们可以使用`ThreadLocal`来实现。因为存入`ThreadLocal`中的数据是和每个线程绑定的，所以不会存在数据竞争的问题了。

#### 使用场景
例子如下：

```java
public class ThreadLocalTest {

    public static void main(String[] args) {
        executeThread();

    }

    private static void executeThread() {
        ExecutorService executor = Executors.newFixedThreadPool(10);
        executor.submit(new MyThread());
        executor.submit(new MyThread());
        executor.submit(new MyThread());
        executor.shutdown();

    }

    static class MyThread extends Thread {
        ThreadLocal threadLocal = new ThreadLocal();
        public void run() {
            String str = Thread.currentThread() + "_" + Math.random();
            threadLocal.set(str);
            try {
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println(threadLocal.get());
        }
    }
}
```

<!--more-->
#### 具体实现
##### set方法
```java
	public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }
```
可以看到这里面的逻辑很简单，得到当前线程，根据当前线程获取map，如果map存在，则直接替换，如果不存在则去初始化map并且放入其中。这里核心的代码是在初始化map时`createMap`方法和`map.set()`，我们进入方法其中看一下这两个方法。


```java
	ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
            table = new Entry[INITIAL_CAPACITY];
            int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
            table[i] = new Entry(firstKey, firstValue);
            size = 1;
            setThreshold(INITIAL_CAPACITY);
    }
```
可以看到`createMap`中最后调用了`ThreadLocalMap`类初始化，这个类看起来还是很简单的和HashMap类似，先建立一个数组，然后对key值取hash值并且对数组长度取与计算，求出在数组中的位置，然后把对象放入数组位置，最后设置阀值。

```java
	private void set(ThreadLocal<?> key, Object value) {

            // We don't use a fast path as with get() because it is at
            // least as common to use set() to create new entries as
            // it is to replace existing ones, in which case, a fast
            // path would fail more often than not.

            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1);

            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                ThreadLocal<?> k = e.get();

                if (k == key) {
                    e.value = value;
                    return;
                }

                if (k == null) {
                    replaceStaleEntry(key, value, i);
                    return;
                }
            }

            tab[i] = new Entry(key, value);
            int sz = ++size;
            if (!cleanSomeSlots(i, sz) && sz >= threshold)
                rehash();
        }
```
set方法和`HashMap`的put方法类似，只不过是这个没有链表了。会先在原有数组中循环寻找当前key，如果找到，则赋新值，返回值。如果在循环过程中，遇到有null的，还会顺便清理掉。如果在数组中没有找到，就证明是新的。把在上面求到的`i`下标值，设置为新的对象。设置之后如果到达阀值会进行一系列的扩容操作，这里就不细说了，因为和HashMap类似。


##### get方法
```java
	public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }
```
这里的核心方法是`map.getEntry(this)`和`setInitialValue()`，后者的方法和上面的set方法类似是为了初始化map就不细说了，这里我们主要说下`map.getEntry(this)`。

```java
		private Entry getEntry(ThreadLocal<?> key) {
            int i = key.threadLocalHashCode & (table.length - 1);
            Entry e = table[i];
            if (e != null && e.get() == key)
                return e;
            else
                return getEntryAfterMiss(key, i, e);
        }
```
可以看到这里主要就是在数组中寻找，如果能直接找到则直接返回，如果不能直接找到，则尝试去当前下标之后找，具体的查找方法请看`getEntryAfterMiss`个人理解的意思就是去不断的查找并且顺便清理为空的，如果找不到返回null。


##### remove方法
```java
	public void remove() {
         ThreadLocalMap m = getMap(Thread.currentThread());
         if (m != null)
             m.remove(this);
     }
```
其实到这里我们已经发现了，ThreadLocal主要的操作都是在对ThreadLocalMap的操作。这里同理，核心的代码是`m.remove(this)`，也就是map中的remove方法。

```java
	private void remove(ThreadLocal<?> key) {
            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1);
            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                if (e.get() == key) {
                    e.clear();
                    expungeStaleEntry(i);
                    return;
                }
            }
        }
```
可以看到这里的方法其实也很简单，就是根据hashcode算出的下标开始在数组中查找，如果找到删除的对象，直接设置为null，这里我个人理解是这样的因为它放到数组的`Entry`对象是继承`WeakReference`的，所以当设置为null之后，垃圾回收器在回收时会优先回收。这里是个人理解的，如有不正确请各位指正。

#### Tips
我们看下来`ThreadLocal`的代码，其实就是对数组的操作进行了封装，所以说如果有大量的写的话会产生和`ArrayList`类似的问题，就是不断的扩容。所以在使用时应该注意，不要有太大量的写入，并且在这里也没有看到像`ArrayList`那样可以指定长度的地方。以上就是个人对此类的理解，如果有不正确之处请大家指正共同进步。




























