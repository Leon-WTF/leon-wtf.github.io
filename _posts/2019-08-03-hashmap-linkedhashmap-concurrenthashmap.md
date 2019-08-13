---
title: "HashMap & LinkedHashMap & ConcurrentHashMap"
category: Java
tag: java
---
# HashMap #
内部用数组存储内部类Node包装的Key-Value，用Key的哈希值来计算数据在数组中的位置，如果遇到计算出来的位置已经有值，则以链表的方式在后面添加，当链表长度变大时，则将其转为红黑树

```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length; // 如果数组占用率超过默认的75%则扩容至原来的两倍
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
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st， 如果binCount>= 8-1,转换为红黑树，但数组长度小于MIN_TREEIFY_CAPACITY时，优先通过扩容来解决
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
Object的**hashcode**采用native C实现，返回实例内存地址，String的默认实现如下：
```java
/**
 * Returns a hash code for this string. The hash code for a
 * {@code String} object is computed as
 * <blockquote><pre>
 * s[0]*31^(n-1) + s[1]*31^(n-2) + ... + s[n-1]
 * </pre></blockquote>
 * using {@code int} arithmetic, where {@code s[i]} is the
 * <i>i</i>th character of the string, {@code n} is the length of
 * the string, and {@code ^} indicates exponentiation.
 * (The hash value of the empty string is zero.)
 *
 * @return  a hash code value for this object.
 */
public int hashCode() {
    int h = hash;
    if (h == 0 && value.length > 0) {
        char val[] = value;

        for (int i = 0; i < value.length; i++) {
            h = 31 * h + val[i];
        }
        hash = h;
    }
    return h;
}
```
HashMap中计算数组位置的方法如下：
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
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16); // 移位int一半的bit数后异或，为了同时利用上哈希值的高位和低位，保留均匀度，减少哈希冲突
}
tab[i = (tab.length - 1) & hash] // 当B为2的幂次方时，A % B = A & (B - 1)，数组长度取2的幂次方可以使n-1的二进制全为1，与操作就相当于去hash的掩码，因此最大化保留其均匀度
```
另外，Object中**equals**的实现是判断两个Object指针是否指向了同一个内存地址, 同时注释中也建议如果覆盖该方法也同时覆盖**hashCode**方法，为了保证相等的对象也具有相同的哈希值。
```java
/**
 * Indicates whether some other object is "equal to" this one.
 * <p>
 * The {@code equals} method implements an equivalence relation
 * on non-null object references:
 * <ul>
 * <li>It is <i>reflexive</i>: for any non-null reference value
 *     {@code x}, {@code x.equals(x)} should return
 *     {@code true}.
 * <li>It is <i>symmetric</i>: for any non-null reference values
 *     {@code x} and {@code y}, {@code x.equals(y)}
 *     should return {@code true} if and only if
 *     {@code y.equals(x)} returns {@code true}.
 * <li>It is <i>transitive</i>: for any non-null reference values
 *     {@code x}, {@code y}, and {@code z}, if
 *     {@code x.equals(y)} returns {@code true} and
 *     {@code y.equals(z)} returns {@code true}, then
 *     {@code x.equals(z)} should return {@code true}.
 * <li>It is <i>consistent</i>: for any non-null reference values
 *     {@code x} and {@code y}, multiple invocations of
 *     {@code x.equals(y)} consistently return {@code true}
 *     or consistently return {@code false}, provided no
 *     information used in {@code equals} comparisons on the
 *     objects is modified.
 * <li>For any non-null reference value {@code x},
 *     {@code x.equals(null)} should return {@code false}.
 * </ul>
 * <p>
 * The {@code equals} method for class {@code Object} implements
 * the most discriminating possible equivalence relation on objects;
 * that is, for any non-null reference values {@code x} and
 * {@code y}, this method returns {@code true} if and only
 * if {@code x} and {@code y} refer to the same object
 * ({@code x == y} has the value {@code true}).
 * <p>
 * Note that it is generally necessary to override the {@code hashCode}
 * method whenever this method is overridden, so as to maintain the
 * general contract for the {@code hashCode} method, which states
 * that equal objects must have equal hash codes.
 *
 * @param   obj   the reference object with which to compare.
 * @return  {@code true} if this object is the same as the obj
 *          argument; {@code false} otherwise.
 * @see     #hashCode()
 * @see     java.util.HashMap
 */
public boolean equals(Object obj) {
    return (this == obj);
}
```
# 一致性哈希算法 #
在分布式缓存系统中为了解决取模求分配机器在机器增减时带来的大量数据迁移问题，采用了一致性哈希算法。简单来说就是将哈希空间想象成0~$2^{32}$-1的环，将集群中的机器取一个哈希值$h_i$，对数据取哈希值，沿顺时针寻找，将其放在第一个遇到的$h_i$对应的机器上。同时为了避免机器数量较少时数据分布不均匀和机器宕机时其数据全部落在环上下一台机器上，引入了虚拟节点的概念，环上设置多个虚拟点，然后间隔的均匀的选取虚拟点将其映射到实际机器上。在实现上可以利用**TreeMap**的**tailMap**函数来取大于当前数据哈希值的最小$h_i$。
> [面试必备：什么是一致性Hash算法？](https://zhuanlan.zhihu.com/p/34985026)
> [一致性哈希算法](https://blog.csdn.net/piqianming/article/details/79670051)

# LinkedHashMap #
LinkedHashMap是在HashMap的基础上，将数据用双向链表连接起来，数据插入时被放在链表的尾部，如果数据已经存在则将其取出更新再插入在链表尾部，数据读取时也将其取出放入链表的尾部。这天然就可以支持Least Recently Use缓存清除策略的实现。

# ConcurrentHashMap #
它在HashMap的基础上实现了多线程安全，在Java 1.7的版本中主要通过分段锁（ReentrantLock）数组来实现的，在Java 1.8中则利用CAS+Synchronized来实现, 相当于将分段细分到了数组中的每个元素。下面以put函数为例来看下它是如何实现的：
```java
public V put(K key, V value) {
    return putVal(key, value, false);
}
/** Implementation for put and putIfAbsent */
final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
    int hash = spread(key.hashCode());
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        if (tab == null || (n = tab.length) == 0)
        	// 数组初始化，如果sizeCtl<0则代表有线程正在初始化线程，让出CPU使用
            tab = initTable();
        // 通过unsafe.getObjectVolatile获取数组元素，数组在Java中是对象，保存在堆中，仅仅是指向数组首地址的引用是volatile变量，里面存储的对象并不能保证实时更新到了CPU缓存
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
        	 // no lock when adding to empty bin，通过unsafe.compareAndSwapObject更新数组元素，如果不成功则进入下一个for循环
            if (casTabAt(tab, i, null, new Node<K,V>(hash, key, value, null)))
                break;                  
        }
        else if ((fh = f.hash) == MOVED)
        	// 代码数组正在扩容，则帮助完成多线程扩容
            tab = helpTransfer(tab, f);
        else {
            V oldVal = null;
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    if (fh >= 0) {
                        binCount = 1;
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
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
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                       value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }
            if (binCount != 0) {
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    // 记录元素个数，并检查是否需要进行扩容
    addCount(1L, binCount);
    return null;
}
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
    CounterCell[] as; long b, s;
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
            // 将sizeCtl设为负数，保证新建数组只有一个线程来进行
            else if (U.compareAndSwapInt(this, SIZECTL, sc, (rs << RESIZE_STAMP_SHIFT) + 2))
                transfer(tab, null);
            s = sumCount();
        }
    }
}
```
在**transfer**中进行数组扩容，分为创建新数组和从旧数组迁移数据到新数组，其中第二步是可以多线程并行完成，每个线程都将处理完的数组位置附上**ForwardingNode**代表已完成。

> [深入浅出ConcurrentHashMap1.8](https://www.jianshu.com/p/c0642afe03e0)
> [深入分析ConcurrentHashMap1.8的扩容实现](https://www.jianshu.com/p/f6730d5784ad)
