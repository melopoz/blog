---
title: Java_HashMap 不止是HashMap
tags: 
  - JUC
  - 并发
  - 同步
  - java1.8优化
  - hash散列的优化
categories:
  - [Java]
---



# java.util.Map

<img alt="Map接口结构，图在github" src="https://raw.githubusercontent.com/melopoz/pics/master/img/java-util-map.png" style="zoom:50%;" />



## 常用实现

1. HashMap

   根据hashCode存储数据，访问速度快，遍历顺序不确定，允许一个key为null，允许任意value为null，非线程安全，同一时刻多个线程同时对一个hashMap进行put，可能会导致数据不一致。

   > 可用Collections#synchronizedMap方法得到一个线程安全的map（该类直接是对map对象加锁）或者用ConcurrentHashMap

2. HashTable

   遗留类，不建议用，一般也不用这玩意了。常用功能和HashMap类似，不过HashTable是`extends Dictionary implements Map`，线程安全，但并发性不如ConcurrentHashMap，他的方法都是synchronized的。

3. LinkedHashMap

   继承自HashMap，记录了key的插入顺序

   > accessOrder参数：默认false，是根据第一次插入该key的顺序排序；true：每次访问都更新排序，也就是get(k)、再put(k,v)都会将k放到最后。

4. TreeMap

   实现NavigableMap接口(navigable:可导航的)，NavigableMap继承SortedMap接口。

   红黑树结构，Entry根据key排序，默认按照key升序排序，也可以在构造函数中指定用于排序的比较器Comparator。

   > 使用TreeMap时，key必须实现Comparable接口或者在TreeMap的构造函数中传入自定义的Comparator，否则运行时会throw ClassCastException。



## HashMap内部实现

### 数据结构 & 属性

数组 + 链表 / 红黑树(1.8增加红黑树)

<img alt="HashMap数据结构，图在github" src="https://raw.githubusercontent.com/melopoz/pics/master/img/HashMap%E7%BB%93%E6%9E%84.png" style="zoom:50%;" />

#### hash数组：`Node<K, V>[] table；`

哈希桶数组，每个元素都是一个键值对：`Node<K,V> implements Map.Entry`。并维护下一个节点的引用`next`。

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

    public final K getKey()        { return key; }
    public final V getValue()      { return value; }
    public final String toString() { return key + "=" + value; }

    public final int hashCode() {
        // hashCode函数如果参数是null，都返回0，所以key=null的都是存到table[0]
        return Objects.hashCode(key) ^ Objects.hashCode(value);
    }
    ...
}
```

> hash可能会冲突，可以采用开放地址法、链地址法。HashMap采用链地址法。
>
> > 开放地址法：每当目标位置不是空的，就向下寻找(index+1)，直到找到空位置。
>
> 数组size小了，再好的hash函数也会很容易冲突，这种结构必然要权衡空间成本和时间成本，需要用好的hash算法和扩容机制来让table占据空间小，又不容易发生hash碰撞。



#### 数组容量：table.length 默认16

负载因子`loadFactor`默认为`0.75`

阈值`threshold` = `length * loadFactor`，table中的元素个数超过threshold之后就要扩容`resize()`。

> 时间效率要求极高就**减小threshold**，容量大了hash冲突就少了，时间复杂度更趋向于O(1)；
>
> 内存紧张就**调大threshold**，减少table使用的空间。



#### table.length 必须为2的n次方

扩容(resize)就 *2，**也是1.8HashMap的一个优化**。更多见下文[**确定node在table中的索引位置**](#确定node在table中的索引位置)

因为这种根据hash在table中找位置都是用位运算`hash值 & (length-1)`，所以这个掩码的1越多，结果就越均匀。

> 对比HashTable扩容是`* 2 - 1`，初始容量是个素数`11`，但是HashTable扩容后的数字并不理想，扩容后的length二进制标识就三个1:
>
> > 11=8+2+1   `0000 1011`
> >
> > 21=16+4+1   `0001 0101`，相当于每次把高位的`101`左移一位
> >
> > 41=32+8+1   `0010 1001`，找hash对应位置的时候还要把最低位的1减掉..
> >
> > 81=64+16+1   `0101 0001`
>
> 本来mask取素数应该是为了取余(flag&mask)的时候，mask有更多的1，相当于让flag有更多有效位，结果1.7的实现并不理想。

HashMap中length必须是2的n次方，`16`binaryStr: `0001 0000`，减1就直接把唯一一个高电平位的右边全置为1：`16-1`binaryStr：`0000 1111`，这样node在table中的分配就更均匀了。



#### 结构变化次数：modCount

记录hashMap内部结构发生变化的次数，主要用于迭代时的快速失败（fail-fast）。

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

确定元素在table中的位置总的来说，就是hash值对table.length取余，得到一个[0,length)的值，就是那个位置咯。

位运算`hash值 & (length-1)`，也是得到一个[0,length)的值，位运算效率肯定比数学运算要高。

先看看1.7的代码（jdk1.8没有这个方法）

```java
static int indexFor(int h, int length) {
     return h & (length-1);  // 直接取模运算
}
```

再看1.8的优化：hash值 ^ hash值的高16位，这样能在容量比较小的情况下hash值的高位也能影响到node在table中的位置

```java
// 1.求key的hash时，让高位也参与了运算
static final int hash(Object key) {
    int h;
                                     // key.hash值 异或 hash值的高16位
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    // >>>:逻辑右移，忽略符号位，移动完第一位符号位肯定是0，是个正整数了
}

// Object#hashCode
public native int hashCode();// int 32位

// 2.确定位置：flag&mask（hash&(length-1)）  掩码mask:length是2的n次方，减1之后原来的1变为0，右边的位都是1，结果就是[0,length)范围内的数
// 比如resize()方法重新计算位置的时候的代码,前边有介绍了
newTab[e.hash & (newCap - 1)] = e;
```

> > ps：曾经，dalao问我有什么比取余效率更高的，应该就是想听我说`&运算`吧 。
> >
> > 所以不要把`&`说成取余了，它毕竟是个数学的词，对方完全可以认为你说的是`%`。
> >
> > 位运算`hash & (length-1)`肯定比数学运算-取余`hash % (length-1)`效率要高啊。
> >
> > 哦，dalao还问redis怎么让那些key分布更均匀一些，应该想听让高位也参与运算吧，当然还得有好的hash函数。

相关的优化还有一个，就是`resize()`扩容时的 **oldCap作mask 重hash优化**，key的`newIndex`只可能是`oldIndex`或`oldIndex+oldCap`，决定因素就是oldCap这个位`1 还是 0`。见下方[扩容 resize()](扩容 resize())



#### put(K, V)方法

流程：

1. 是否需要初始化？需要就扩容；
2. `table[hash&(length-1)] `是不是`null`？
   - 是`null`：直接插入；
   - 有`node`：key相同吗？
     - 相同：覆盖
     - 不同：`instance of TreeNode` ？`红黑树插入` : `遍历链表覆盖或加到tail.next`,并判断是否需要转红黑树`treeifyBin()`；
3. 最后**都**要再判断size是否超过阈值`threshold`，超过就扩容(扩容会`rehash`哦)。

<img alt="put()流程图，图在github" src="https://raw.githubusercontent.com/melopoz/pics/master/img/HashMap" style="zoom:40%;" />

```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}
final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    if ((tab = table) == null || (n = tab.length) == 0) // table是空的 扩容(初始化扩容)
        n = (tab = resize()).length;
    if ((p = tab[i = (n - 1) & hash]) == null)// key不存在 直接放到对应索引位
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;// p: table[i]处的head节点
        if (p.hash == hash &&
              ((k = p.key) == key || (key != null && key.equals(k))))// 要插入的节点对应table[i]处链表的head 直接走后边的覆盖逻辑
            e = p;
        else if (p instanceof TreeNode)// 是树节点
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {// 否则就遍历链表，找到key相同的节点或者加到tail后边
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {// table[i]处就一个节点
                    p.next = newNode(hash, key, value, null);
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st  
                        treeifyBin(tab, hash); // TREEIFY_THRESHOLD=8，所以如果当前节点是第8个就会转换成红黑树
                    break;
                }
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k)))) // 如果key已经存在 就覆盖
                    break;
                p = e;
            }
        }
        
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);// LinkedHashMap重写了这个方法进行排序，如果设置了accessOrder=true，则访问节点之后节点要放到最后
            return oldValue;
        }
    }
    ++modCount;
    if (++size > threshold) // 超过阈值就扩容
        resize();
    afterNodeInsertion(evict); // 插入节点之后 同上边 afterNodeAccess..
    return null;
}
```



#### 扩容 resize()

table是数组，不能扩容，所以肯定开辟新空间，再复制过去。

由于扩容是旧容量*2：`newCap = oldCap << 1`，key的hash值是固定的，举个例子：(就只看后八位了)

> 假设  hash(key1) = `0000 0101`，hash(key2) = `0001 0101`
>
> oldCap= 16（15的二进制：`0000 1111`）
>
> 这两个key通过`hash(key)&(length-1)`运算得到的位置都是**5** `0101`。
>
> key1对应的位置：`0000 0101` & `0000 1111` = 5（ `0000 0101`）
>
> key2对应的位置：`0001 0101` & `0000 1111` = **5**（ `0000 0101`）

扩容之后：

> newCap：32（31的二进制：`0001 1111`）
>
> key1对应的位置：`0000 0101` & `0001 1111` = 5（ `0000 0101`）
>
> key2对应的位置：`0001 0101` & `0001 1111` = **21**（ `0001 0101`）

<img alt="rehash demo，图在github" src="https://raw.githubusercontent.com/melopoz/pics/master/img/resize-key%E7%9A%84%E6%96%B0%E7%B4%A2%E5%BC%95%E4%BD%8D%E7%BD%AE.png" style="zoom: 50%;" />

这么看来key在新table中的位置只有两种可能：`oldIndex`或者`oldIndex + oldCap`。

其决定因素就是`hash(key)`在`mask的高位新增位(上图红色的1)` 对应的bit是1还是0，这个`binaryStr只有一个1的`mask(`0001 0000`)正是`oldCap`。所以

> `hash & oldCap == `**0**：`newIndex = oldIndex  `
>
> `hash & oldCap == `**1**：`newIndex = oldIndex + oldCap`

> Ctrl+F 搜索`"oldCap作mask 重hash优化"`找到对应的代码  :)

##### resize()源码

```java
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16 // 初始化容量
static final int MAXIMUM_CAPACITY = 1 << 30;// 最大容量
static final float DEFAULT_LOAD_FACTOR = 0.75f;// 默认负载因子
static final int TREEIFY_THRESHOLD = 8;// 树化的阈值，链表超过8个，就转为红黑树（条件一）
static final int UNTREEIFY_THRESHOLD = 6;// 红黑树的节点个数小于6就转为链表
static final int MIN_TREEIFY_CAPACITY = 64;// table的容量低于64不会树化（条件二）

final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    // 上来先 base case ..
    if (oldCap > 0) {
        if (oldCap >= MAXIMUM_CAPACITY) {// MAXIMUM_CAPACITY = 1 << 30;
            threshold = Integer.MAX_VALUE;// 容量超过最大值，把阈值也搞大，这样以后就不用resize()了
            return oldTab;
        }
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&// 容量 *2
                 oldCap >= DEFAULT_INITIAL_CAPACITY)// DEFAULT_INITIAL_CAPACITY = 1 << 4; // 16
            newThr = oldThr << 1; // double threshold // 阈值也 *2
    }
    else/*oldCap==0*/ if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;// 初始容量设置为阈值
    else/*oldCap==0 && oldThr == 0*/ { // zero initial threshold signifies using defaults
        // 没有设置初始阈值，就使用默认值
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    if (newThr == 0) { // 如果newThr为0，那直接设置容量为Integer.MAX
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    // 从这里开始
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];// 开辟newTab空间
    table = newTab;
    if (oldTab != null) {
        for (int j = 0; j < oldCap; ++j) {// 遍历oldTable
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;// 去掉以前的引用 help GC
                if (e.next == null)// 说明这个位置原本只有一个node
                    newTab[e.hash & (newCap - 1)] = e;// 放到newTable对应的位置即可
                else if (e instanceof TreeNode)// 是红黑树节点
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);// todo 红黑树操作 
                else /*这个位置是链表*/{ // preserve order
                    Node<K,V> loHead = null, loTail = null;// 假设容量是从16扩容到32，loHead就是newTable[15]这个head节点
                    Node<K,V> hiHead = null, hiTail = null;// 那么hiHead就是newTable[31]这个head节点
                    Node<K,V> next;
                    do {// 遍历这个链表
                        next = e.next;// 先留一份当前node的next的指针,while的时候用
                        // 这里就是"oldCap作mask 重hash优化"中对 newIndex只有两种可能 的处理
                        if ((e.hash & oldCap) == 0) {// 在newTable中的位置还是 oldIndex
                            if (loTail == null)// 第一次来newTable[j]这个位置肯定是null
                                loHead = e;// loHead指针直接指向这个新node
                            else
                                loTail.next = e;// 第二次之后插入到table[j]对应链表的tail之后，可见1.8这里是用的尾插法。
                            loTail = e;// 无论如何总是要更新tail节点指向新的node
                        }
                        else {// 在newTable中的位置是 oldIndex + oldCap，同上
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    // do-while结束，newTable[j]和newTable[j+oldCap]对应的链表已经生成，放到newTable即可
                    if (loTail != null) {// 说明newTable[j]这个位置有节点
                        loTail.next = null;
                        newTab[j] = loHead;// loHead放到newTable的低索引位
                    }
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;// hiHead放到newTable的高索引位
                    }
                }
            }
        }
    }
    return newTab;
}
```

再来看看1.7的源码

```java
void transfer (Entry[] newTable){
    Entry[] src = table;//old table
    int newCapacity = newTable.length;
    for (int j = 0; j < src.length; j++) {
        Entry<K, V> e = src[j];// e就是旧table lo索引位的 head节点
        if (e != null) {
            src[j] = null;//释放旧Entry数组的对象引用 help GC
            do {
                Entry<K, V> next = e.next;// 持有一份当前node的next指针
                int i = indexFor(e.hash, newCapacity); // 重新计算每个元素在数组中的位置 还是用的 e.hash & (newCapacity-1)
                e.next = newTable[i]; // 将newTable[i]处的链表接到当前node(新head)后边。  - 头插法！死循环
                newTable[i] = e;      // 将当前node放到newTable[i]处
                e = next;
            } while (e != null);
        }
    }
}
static int indexFor(int h, int length) {
     return h & (length-1);
}
```

1.8用的是尾插法，链表元素不会倒置，1.7的代码中用的是头插法，元素位置会倒置，而且可能死循环



#### 树化操作 treeifyBin() - 两个条件

如果`hash槽的node个数>=8`，就会调用treeifyBin(..)方法进行树化。

```java
if (binCount >= TREEIFY_THRESHOLD)
    treeifyBin(tab, i);
```

treeifyBin(..)方法中会再次判断，**只有table的容量`table.length`>=64**，才会转为红黑树，否则只是扩容`resize()`。

```java
final void treeifyBin(Node<K,V>[] tab, int hash) {
    int n, index; Node<K,V> e;
    if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY) // 如果元素个数 < 64 会扩容而不是转换成红黑树。
        resize();
    else if ((e = tab[index = (n - 1) & hash]) != null) {
        TreeNode<K,V> hd = null, tl = null;
        do {
            TreeNode<K,V> p = replacementTreeNode(e, null);
            if (tl == null)
                hd = p;
            else {
                p.prev = tl;
                tl.next = p;
            }
            tl = p;
        } while ((e = e.next) != null);
        if ((tab[index] = hd) != null)
            hd.treeify(tab);
    }
}

TreeNode<K,V> replacementTreeNode(Node<K,V> p, Node<K,V> next) {
    return new TreeNode<>(p.hash, p.key, p.value, next);
}
```

```
replacementTreeNode
```





### 常见问题：

###### 扩容的时间复杂度是多少

> O(logn) ~ O(n)
>
> 肯定要要遍历所有node，如果每个节点都是红黑树，那就是cap*log(n)，去掉常量就是O(logn)，如果都是链表，相当于遍历cap次。
>
> (注意不会因为for嵌套do-while就是O(²))。

###### 一些细节 特点

> 扩容非常耗时，尽量初始化的时候预估容量。
>
> 负载因子loadFactor是可以指定的，不是特殊情况不要改。

###### 1.8做了哪些优化

> 1. 加入红黑树；
> 2. 头插法换成尾插法，和红黑树结构均能防止死循环；
> 3. 求key对应的索引位置时用 oldCap 做mask 直接判断 node 对应的索引是 oldIndex 还是 oldIndex + oldCap。

###### 为啥用红黑树？ 

> 如果hash函数结构够均匀，性能大概是提升15%。

> 如果hash函数很垃圾，比如极端情况所有hash(key)都相同。那每次都要遍历table[j]处对应的链表，1.7在这种情况下时间复杂度直接变成O(n)，1.8引入红黑树，在这种情况下节点越多查询节点可能越小，时间复杂度下降为O(logn)。

###### 怎么解决1.7那个死循环的？

> 1.7是把node放到newTable[j]处的head（头插法），1.8是放到newTable[j]的tail.next（尾插法）。即使线程不安全，出问题也就是可能node会有重复





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