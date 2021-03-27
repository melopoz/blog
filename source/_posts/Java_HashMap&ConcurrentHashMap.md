---
title: Java_HashMap & ConcurrentHashMap
tags: 
  - JUC
  - 并发
  - 同步
categories:
  - [Java,JUC]
  - [Java]
---



# HashMap

### HashMap 和 Hashtable 的区别

|                 | HashMap                     | HashTable                                   |
| --------------- | --------------------------- | ------------------------------------------- |
| key,value为null | 允许，且只能有一个key为null | 不允许                                      |
| 默认初始容量    | 16                          | 11                                          |
| 扩容            | 原容量的2倍                 | 原来的2倍+1                                 |
|                 | 重                          | 直接使用hashCode()                          |
| 线程安全        | 非线程安全                  | 线程安全，就是HashMap的方法加个synchronized |
|                 |                             |                                             |
|                 |                             |                                             |
|                 |                             |                                             |
|                 |                             |                                             |
|                 |                             |                                             |

HashMap 允许 key 和 value 为 null，Hashtable 不允许。
HashMap 的默认初始容量为 16，Hashtable 为 11。
HashMap 的扩容为原来的 2 倍，Hashtable 的扩容为原来的 2 倍加 1。
HashMap 是非线程安全的，Hashtable是线程安全的。
HashMap 的 hash 值重新计算过，Hashtable 直接使用 hashCode。
HashMap 去掉了 Hashtable 中的 contains 方法。
HashMap 继承自 AbstractMap 类，Hashtable 继承自 Dictionary 类。



### 数据结构

HashMap 的底层是个 Node 数组（Node<K,V>[] table），在数组的具体索引位置，如果存在多个节点，则可能是以链表或红黑树的形式存在。

### api及实现

增加、删除、查找，第一步都要定位到key在hashTable中的位置。

> 1）拿到`key`的`hashCode`值；
>
> 2）将`hashCode的高位(前16位)` 进行异或`(^)`，重新计算hash值；
>
> ```java
> static final int hash(Object key) {
>     int h;
>     return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
> }
> ```
>
> 3）将`新hash值`和`table.length-1`进行与运算 找到该位置。

该位置存放的是链表的`head`节点。如果该接单



HashMap 的默认初始容量（capacity）是 16，capacity 必须为 2 的幂次方；默认负载因子（load factor）是 0.75；实际能存放的节点个数（threshold，即触发扩容的阈值）= capacity * load factor。
HashMap 在触发扩容后，阈值会变为原来的 2 倍，并且会对所有节点进行重 hash 分布，重 hash 分布后节点的新分布位置只可能有两个：“原索引位置” 或 “原索引+oldCap位置”。例如 capacity 为16，索引位置 5 的节点扩容后，只可能分布在新表 “索引位置5” 和 “索引位置21（5+16）”。
导致 HashMap 扩容后，同一个索引位置的节点重 hash 最多分布在两个位置的根本原因是：1）table的长度始终为 2 的 n 次方；2）索引位置的计算方法为 “(table.length - 1) & hash”。HashMap 扩容是一个比较耗时的操作，定义 HashMap 时尽量给个接近的初始容量值。
HashMap 有 threshold 属性和 loadFactor 属性，但是没有 capacity 属性。初始化时，如果传了初始化容量值，该值是存在 threshold 变量，并且 Node 数组是在第一次 put 时才会进行初始化，初始化时会将此时的 threshold 值作为新表的 capacity 值，然后用 capacity 和 loadFactor 计算新表的真正 threshold 值。
当同一个索引位置的节点在增加后达到 9 个时，并且此时数组的长度大于等于 64，则会触发链表节点（Node）转红黑树节点（TreeNode），转成红黑树节点后，其实链表的结构还存在，通过 next 属性维持。链表节点转红黑树节点的具体方法为源码中的 treeifyBin 方法。而如果数组长度小于64，则不会触发链表转红黑树，而是会进行扩容。
当同一个索引位置的节点在移除后达到 6 个时，并且该索引位置的节点为红黑树节点，会触发红黑树节点转链表节点。红黑树节点转链表节点的具体方法为源码中的 untreeify 方法。
HashMap 在 JDK 1.8 之后不再有死循环的问题，JDK 1.8 之前存在死循环的根本原因是在扩容后同一索引位置的节点顺序会反掉。
HashMap 是非线程安全的，在并发场景下使用 ConcurrentHashMap 来代替。

>  原理  https://blog.csdn.net/v123411739/article/details/78996181

> hashmap 在**jdk1.7**中可能出现**死循环**。具体是在resize扩容的时候，因为扩容的的时候会重新计算元素的位置，如果出现环形链表，就会导致死循环，cpu100%；
>
> https://www.jianshu.com/p/4930801e23c8

> jdk**1.8**中 死循环 的原因有一种是 **两个TreeNode节点的parent节点都指向对方**。
>
> https://blog.csdn.net/qq_33330687/article/details/101479385



常见问题：

为啥用红黑树？ 

怎么解决1.7那个死循环的？。。。想起这问题我就难受



# ConcurrentHashMap

>  key和value不能为null，key不能，value不能因为多线程使用，有歧义

jdk1.7 使用segment分段锁

### jdk1.8

get()

> 使用volatile修饰变量，直接get，不会获取到旧值。
>
> ```java
> public V get(Object key) {
>         Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
>         int h = spread(key.hashCode());//散列 得到key在数组中的位置
>         if ((tab = table) != null && (n = tab.length) > 0 &&
>             (e = tabAt(tab, (n - 1) & h)) != null) {
>             if ((eh = e.hash) == h) {
>                 if ((ek = e.key) == key || (ek != null && key.equals(ek)))
>                     return e.val;
>             }
>             else if (eh < 0)
>                 return (p = e.find(h, key)) != null ? p.val : null;
>             while ((e = e.next) != null) {
>                 if (e.hash == h &&
>                     ((ek = e.key) == key || (ek != null && key.equals(ek))))
>                     return e.val;
>             }
>         }
>         return null;
>     }
> @SuppressWarnings("unchecked")
> static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
>     return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
> }
> ```
>
> 

put()

> 使用cas和synchronized：
>
> ```java
> public V put(K key, V value) {
>     return putVal(key, value, false);
> }
> 
> /** Implementation for put and putIfAbsent */
> final V putVal(K key, V value, boolean onlyIfAbsent) {
>     if (key == null || value == null) throw new NullPointerException();
>     int hash = spread(key.hashCode());
>     int binCount = 0;
>     for (Node<K,V>[] tab = table;;) {
>         Node<K,V> f; int n, i, fh;
>         if (tab == null || (n = tab.length) == 0)
>             tab = initTable();
>         else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
>             // put到空容器中 不需要锁 cas即可
>             if (casTabAt(tab, i, null,
>                          new Node<K,V>(hash, key, value, null)))
>                 break;   // no lock when adding to empty bin
>         }
>         else if ((fh = f.hash) == MOVED)
>             // 如果正在调整大小，则帮助转移
>             tab = helpTransfer(tab, f);
>         else {
>             V oldVal = null;
>             // 否则就是添加新元素。需要synchronized 获取当前node的锁
>             synchronized (f) {
>                 if (tabAt(tab, i) == f) {
>                     if (fh >= 0) {
>                         binCount = 1;
>                         for (Node<K,V> e = f;; ++binCount) {
>                             K ek;
>                             if (e.hash == hash &&
>                                 ((ek = e.key) == key ||
>                                  (ek != null && key.equals(ek)))) {
>                                 oldVal = e.val;
>                                 if (!onlyIfAbsent)
>                                     e.val = value;
>                                 break;
>                             }
>                             Node<K,V> pred = e;
>                             if ((e = e.next) == null) {
>                                 pred.next = new Node<K,V>(hash, key,
>                                                           value, null);
>                                 break;
>                             }
>                         }
>                     }
>                     else if (f instanceof TreeBin) {
>                         Node<K,V> p;
>                         binCount = 2;
>                         if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
>                                                               value)) != null) {
>                             oldVal = p.val;
>                             if (!onlyIfAbsent)
>                                 p.val = value;
>                         }
>                     }
>                 }
>             }
>             if (binCount != 0) {
>                 if (binCount >= TREEIFY_THRESHOLD)
>                     treeifyBin(tab, i);
>                 if (oldVal != null)
>                     return oldVal;
>                 break;
>             }
>         }
>     }
>     addCount(1L, binCount);
>     return null;
> }
> 
> @SuppressWarnings("unchecked")
> static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
>     return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
> }
> 
> // 使用cas操作
> static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i,
>                                     Node<K,V> c, Node<K,V> v) {
>     return U.compareAndSwapObject(tab, ((long)i << ASHIFT) + ABASE, c, v);
> }
> ```
>
> 





md 又是 todo