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



# java.util.Map

<img src="https://raw.githubusercontent.com/melopoz/pics/master/img/java-util-map.png" style="zoom:50%;" />



## 常用实现

1. HashMap

   根据hashCode存储数据，访问速度快，遍历顺序不确定，允许一个key为null，允许任意value为null，非线程安全，同一时刻多个线程同时对一个hashMap进行put，可能会导致数据不一致。

   > 可用Collections#synchronizedMap方法得到一个线程安全的map（直接synchronized入参的map对象）
   >
   > 或者用ConcurrentHashMap

2. HashTable

   遗留类，不建议用，一般也不用这玩意了。常用功能和HashMap类似，不过HashTable是`extends Dictionary implements Map`，线程安全，但并发性不如ConcurrentHashMap，他的方法是synchronized的。

3. LinkedHashMap

   继承自HashMap，记录了key的插入顺序

   > accessOrder参数：默认false，是根据第一次插入该key的顺序排序；true：每次访问都更新排序，也就是get(k)、再put(k,v)都会将k放到最后。

4. TreeMap

   实现NavigableMap接口(navigable:可导航的)，NavigableMap继承SortedMap接口。Entry根据key排序，默认按照key升序排序，也可以指定用于排序的比较器。

   > 使用TreeMap时，key必须实现Comparable接口或者在TreeMap的构造函数中传入自定义的Comparator，否则运行时会throw ClassCastException。



## HashMap内部实现

### 数据结构 & 属性

数组 + 链表 / 红黑树(1.8增加红黑树)

<img src="https://raw.githubusercontent.com/melopoz/pics/master/img/HashMap%E7%BB%93%E6%9E%84.png" style="zoom:50%;" />

#### `Node<K, V>[] table；`

哈希桶数组，每个元素都是一个键值对：`Node<K,V> implements Map.Entry`。

> hash可能会冲突，可以采用开放地址法、链地址法。HashMap采用链地址法。
>
> > 开放地址法：每当目标位置不是空的，就像下寻找，直到找到空位置。
>
> 这种结构必然要权衡空间成本和时间成本，需要用好的hash算法和扩容机制来让table占据空间小，又不容易发生hash碰撞。
>
> table默认长度`length`为`16`，负载因子`loadFactor`为`0.75`。
>
> 阈值`threshold` = `length * loadFactor`，table中的元素个数超过threshold之后就要resize()。
>
> > 时间效率要求极高就减小threshold，内存紧张就调大threshold。

#### table.length

必须为2的n次方，扩容(resize)就 *2

> 因为他用key的hash值对（length-1）取模（&）。**为了减少hash冲突**。可以看resize()的代码和下文**确定node在table中的索引位置**。
>
> ```java
> final Node<K,V>[] resize() {
>         Node<K,V>[] oldTab = table;// 旧table
>         int oldCap = (oldTab == null) ? 0 : oldTab.length;// 旧容量
>         int oldThr = threshold;// 旧阈值
>         int newCap, newThr = 0;// 新容量 新阈值
>         if (oldCap > 0) {
>         	...
>             else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
>                      oldCap >= DEFAULT_INITIAL_CAPACITY)
>                 // 新容量 = 旧容量 * 2
>                 newThr = oldThr << 1; // double threshold
>         }
>         ...
>         if (oldTab != null) {
>             for (int j = 0; j < oldCap; ++j) {// 把entry放到新数组对应的位置
>                 newTab[e.hash & (newCap - 1)] = e;// 用entry.key的hash对 新容量-1 取模，这样就能用上所有的bit
>                 // 举个例子 16-1=15，二进制为`.. 0000 1111`，这样有效位是后四位全部，而且还在容量范围之内。
>         ...
> }
> ```
>
> 对比HashTable来看，hashTable扩容是`*2-1`，初始容量是个素数`11`，但是扩容后就不一定了。
>
> 一般来说容量是素数更能减少hash冲突（如果直接对length取模）（二进制的情况下1多0少），hashMap容量是2，但是他取模的时候巧妙的用的`length-1`，这样同样也是为了二进制下1多0少，让hash值&运算的时候能有更多有效bit。
>
> > ps：Ali，DiDi面试好像都问到这个了，当时由于脑子空白，也忘了回答的啥了 T.T。
> >
> > > 啊！我想起来了，dalao问有什么比取余效率更高的。。其实就是让hash & (length-1)，位运算-&运算肯定比数学运算-%取余 效率要高啊。
> > >
> > > 我以后应该不会说`取模/模以length`了，我以为我在说 & 运算，其实对方可能以为我说的是数学的取余。T.T
> > >
> > > 突然想起高中数学老师如果改行敲代码应该也会是个大佬...

#### modCount

记录hashMap内部结构发生变化的次数，主要用于迭代的快速失败（fail-fast）。

> put新Entry算，put已存在的key(覆盖)不算。

> 可以看`java.util.HashMap.HashIterator#nextNode`的代码，很简单。
>
> ```java
> abstract class HashIterator {
>     Node<K,V> next;        // next entry to return
>     Node<K,V> current;     // current entry
>     int expectedModCount;  // for fast-fail
>     int index;             // current slot
> 
>     HashIterator() {
>         expectedModCount = modCount;// 开始迭代的时候 map实例的修改次数(其实可以理解为版本号嘛)
>         Node<K,V>[] t = table;
>         current = next = null;
>         index = 0;
>         if (t != null && size > 0) { // advance to first entry
>             do {} while (index < t.length && (next = t[index++]) == null);
>         }
>     }
> 
>     public final boolean hasNext() {
>         return next != null;
>     }
> 
>     final Node<K,V> nextNode() {
>         Node<K,V>[] t;
>         Node<K,V> e = next;
>         if (modCount != expectedModCount)// 迭代的时候如果map的结构被改动过
>             throw new ConcurrentModificationException();// 熟悉的exception
>         if (e == null)
>             throw new NoSuchElementException();
>         if ((next = (current = e).next) == null && (t = table) != null) {
>             do {} while (index < t.length && (next = t[index++]) == null);
>         }
>         return e;
>     }
> 
>     public final void remove() {// 所以推荐用iter.remove()
>         Node<K,V> p = current;
>         if (p == null)
>             throw new IllegalStateException();
>         if (modCount != expectedModCount)
>             throw new ConcurrentModificationException();
>         current = null;
>         K key = p.key;
>         removeNode(hash(key), key, null, false, false);
>         expectedModCount = modCount;// 会更新期望版本号
>     }
> }
> ```
>
> `java.util.ListIterator`的代码也是如此，调用迭代器的iterator的remove()方法会更新`modCount`和`expectedModCount`，不会`throw new ConcurrentModificationException()`。



### 功能实现

#### 确定node在table中的索引位置

确定元素在table中的位置总的来说，就是`hash值 & (length-1)`，得到一个0~length的位置。

```java
// 求key的hash，让高位也参与了运算
static final int hash(Object key) {
    int h;
    // hash值 异或 hash值的高位(前16位)
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);// 逻辑右移，移动完第一位符号位肯定是0，是个正整数了
}
// Object#hashCode
public native int hashCode();// int 32位
// 比如resize()方法重新计算位置的时候,前边有介绍了
// newTab[e.hash & (newCap - 1)] = e;
```



#### put(K, V)方法

流程就是这样，就是判断是否扩容，改index一共有多少个Node，多了就红黑树，少了就链表。所以就记一下触发这些操作的条件吧

<img src="https://raw.githubusercontent.com/melopoz/pics/master/img/HashMap" style="zoom:40%;" />

#### 扩容 resize()

table是数组，不能扩容，所以肯定开辟新空间，再复制过去。

由于扩容是旧容量*2：`newCap = oldCap << 1`，key的hash值是固定的，举个例子：(就只看后八位了)

> 假设  hash(key1) = `0000 0101`，hash(key2) = `0001 0101`
>
> oldCap= 16（15的二进制：`0000 1111`）
>
> 这两个key通过`hash(key)&(length-1)`运算得到的位置都是5 `0101`。

扩容之后：

> newCap：32（31的二进制：`0001 1111`）
>
> key1对应的位置：`0000 0101` & `0001 1111` = 5（ `0000 0101`）
>
> key2对应的位置：`0001 0101` & `0001 1111` = 21（ `0001 0101`）

所以

#### 链表 还是 红黑树



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

该位置存放的是链表的`head`节点。如果该节点



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
>      Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
>      int h = spread(key.hashCode());//散列 得到key在数组中的位置
>      if ((tab = table) != null && (n = tab.length) > 0 &&
>          (e = tabAt(tab, (n - 1) & h)) != null) {
>          if ((eh = e.hash) == h) {
>              if ((ek = e.key) == key || (ek != null && key.equals(ek)))
>                  return e.val;
>          }
>          else if (eh < 0)
>              return (p = e.find(h, key)) != null ? p.val : null;
>          while ((e = e.next) != null) {
>              if (e.hash == h &&
>                  ((ek = e.key) == key || (ek != null && key.equals(ek))))
>                  return e.val;
>          }
>      }
>      return null;
>  }
> @SuppressWarnings("unchecked")
> static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
>  return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
> }
> ```
>
> 

put()

> 使用cas和synchronized：
>
> ```java
> public V put(K key, V value) {
>  return putVal(key, value, false);
> }
> 
> /** Implementation for put and putIfAbsent */
> final V putVal(K key, V value, boolean onlyIfAbsent) {
>  if (key == null || value == null) throw new NullPointerException();
>  int hash = spread(key.hashCode());
>  int binCount = 0;
>  for (Node<K,V>[] tab = table;;) {
>      Node<K,V> f; int n, i, fh;
>      if (tab == null || (n = tab.length) == 0)
>          tab = initTable();
>      else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
>          // put到空容器中 不需要锁 cas即可
>          if (casTabAt(tab, i, null,
>                       new Node<K,V>(hash, key, value, null)))
>              break;   // no lock when adding to empty bin
>      }
>      else if ((fh = f.hash) == MOVED)
>          // 如果正在调整大小，则帮助转移
>          tab = helpTransfer(tab, f);
>      else {
>          V oldVal = null;
>          // 否则就是添加新元素。需要synchronized 获取当前node的锁
>          synchronized (f) {
>              if (tabAt(tab, i) == f) {
>                  if (fh >= 0) {
>                      binCount = 1;
>                      for (Node<K,V> e = f;; ++binCount) {
>                          K ek;
>                          if (e.hash == hash &&
>                              ((ek = e.key) == key ||
>                               (ek != null && key.equals(ek)))) {
>                              oldVal = e.val;
>                              if (!onlyIfAbsent)
>                                  e.val = value;
>                              break;
>                          }
>                          Node<K,V> pred = e;
>                          if ((e = e.next) == null) {
>                              pred.next = new Node<K,V>(hash, key,
>                                                        value, null);
>                              break;
>                          }
>                      }
>                  }
>                  else if (f instanceof TreeBin) {
>                      Node<K,V> p;
>                      binCount = 2;
>                      if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
>                                                            value)) != null) {
>                          oldVal = p.val;
>                          if (!onlyIfAbsent)
>                              p.val = value;
>                      }
>                  }
>              }
>          }
>          if (binCount != 0) {
>              if (binCount >= TREEIFY_THRESHOLD)
>                  treeifyBin(tab, i);
>              if (oldVal != null)
>                  return oldVal;
>              break;
>          }
>      }
>  }
>  addCount(1L, binCount);
>  return null;
> }
> 
> @SuppressWarnings("unchecked")
> static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
>  return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
> }
> 
> // 使用cas操作
> static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i,
>                                  Node<K,V> c, Node<K,V> v) {
>  return U.compareAndSwapObject(tab, ((long)i << ASHIFT) + ABASE, c, v);
> }
> ```
>
> 





md 又是 todo