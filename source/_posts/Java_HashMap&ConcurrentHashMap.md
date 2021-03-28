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



### 功能 & 实现

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



#### resize() 扩容

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



#### treeifyBin() 转红黑树 - 两个条件

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

## JDK1.7

### 数据结构

<img alt="jdk1.7-ConcurrentHashMap数据结构，图在github" src="https://raw.githubusercontent.com/melopoz/pics/master/img/926638-20170809132445011-2033999443.png" style="zoom: 67%;" />

#### Segment数组 & HashEntry数组

每个Segment是一个HashEntry数组，每个HashEntry的结构和HashMap相同。

> 用分段锁把锁的粒度变细，以提高并发性。

Segment数组和HashEntry数组的容量都是2的n次方；

`segment数组初始容量16`，只能用16位二进制标识，所以`最大值65536`；

HashEntry`初始容量为1，最小容量为2`。



### 功能 & 实现

```java
// Segment是继承自ReentrantLock的
static class Segment<K,V> extends ReentrantLock implements Serializable {...}
```



#### put(K, V)

1. 如果该Segment还未初始化，就CAS进行初始化；

2. 二次hash，从HashEntry数组中找到key对应的HashEntry

   用tryLock()加锁，失败会自旋，自旋超过指定次数就挂起当前线程，等待被唤醒。



#### get(K)

和HashMap的区别就是需要两次hash，并对Segment加锁



#### size()

在thread1计算size的时候，其他线程可能在thread1计算过的segment插入过数据。

1. 先不加锁，多次计算size，比较计算的结果和前一次的结果，如果一致就返回（最多三次）
2. 如果不一致就把所有segment加锁再计算

代码如下：

```java
try {
    for (;;) {
        if (retries++ == RETRIES_BEFORE_LOCK) {// RETRIES_BEFORE_LOCK 加锁之前重试次数 3
            for (int j = 0; j < segments.length; ++j)
                ensureSegment(j).lock(); // force creation
        }
        sum = 0L;
        size = 0;// size
        overflow = false;
        for (int j = 0; j < segments.length; ++j) {
            Segment<K,V> seg = segmentAt(segments, j);
            if (seg != null) {
                sum += seg.modCount; 
                int c = seg.count; 
                if (c < 0 || (size += c) < 0)
                	overflow = true;// 溢出
            }
        }
        if (sum == last)// 计算过程中结构没有变过
            break;// 只有锁定所有segment 或者 真的没有别的线程改变过map的结构，才会到这里
        last = sum;// 保存本次计算size 的结果，用于下次计算完size的比较 
    }
} finally {
    if (retries > RETRIES_BEFORE_LOCK) {// 如果重试次数超过3，肯定都加上锁再计算的
        for (int j = 0; j < segments.length; ++j)// 所有segment都解锁
            segmentAt(segments, j).unlock();
    }
}
```





## JDK1.8

### 数据结构 & 属性

取消了segments字段，直接使用`transient volatile Node<K,V>[] table`保存数据，与JDK1.8的HashMap类似。

> 保留了Segment的数据结构，但是简化了属性，只是兼容旧版本



##### Node<K,V>[]  table

```java
// 存放node的数组
transient volatile Node<K,V>[] table;

// 和HashMap的Node有些差异，不允许修改Node的value了
static class Node<K,V> implements Map.Entry<K,V> {
    ...
    public final V setValue(V value) {
        throw new UnsupportedOperationException();// 和HashMap的Node相似但是只允许查找，不允许修改
    }
}
```



##### int sizeCtl   -控制标识符

用来控制table的初始化和扩容的操作，不同的值有不同的含义。

- 当为负数时：-1代表正在初始化；-N代表有N-1个线程正在 进行扩容
- 当为0时：代表当时的table还没有被初始化
- 当为正数时：表示`初始化时/触发扩容时`的阈值（size达到这个值就要扩容）

> 也可以只分为`<0`和`>=0`两种情况，因为=0的时候也相当于是个阈值，触发扩容，不过这个扩容是初始化

```java
// 扩容时sizeCtl有两部分组成，第一部分是扩容戳，占据sizeCtl的高16位;第二部分是参与扩容的线程数，占低16位。
// 每个新线程协助扩容时sizeCtl+1，直到所有的低有效位被占满，低有效位默认占16位（最高位为符号位），所以扩容线程数默认最大为65535。
private transient volatile int sizeCtl;
```



##### long baseCount  -基础计数器

```java
// 代表Map中元素个数的的基础计数器，当无竞争时直接使用CAS方式更新该数值 
transient volatile long baseCount;

// 存储Map中元素的计数器，当并发量较高时`baseCount`竞争较为激烈，更新效率较低，所以把变化的数值更新到`counterCells`中的某个节点上
// 计算size()时需要统计每个`counterCells`的大小再加上`baseCount`的数值。
transient volatile CounterCell[] counterCells;

// 扩容或者创建CounterCells时使用的自旋锁（使用CAS实现）；
transient volatile int cellsBusy;
```



##### 常量

```java
static final int MOVED     = -1; // hash for forwarding nodes 转移节点的hash都是-1
static final int TREEBIN   = -2; // hash for roots of trees  红黑树节点hash都是-2
static final int RESERVED  = -3; // hash for transient reservations
static final int HASH_BITS = 0x7fffffff; // usable bits of normal node hash 普通hash的有效位 相当于最基本的掩码吧
```



##### 扩容相关的变量/属性

```java
// 也就是newTable（仅在扩容的时候不为null）
private transient volatile Node<K,V>[] nextTable;

// 扩容子任务对应的table的索引位
transient volatile int transferIndex; 

// 最小转移步长：由于在扩容过程中，会把一个待转移的数组分为多个区间段（转移步长-stride）
private static final int MIN_TRANSFER_STRIDE = 16;

// 可用处理器数量 用来计算扩容任务的步长
static final int NCPU = Runtime.getRuntime().availableProcessors();

// 因为要使用乐观的cas更新变量来保证线程安全性，肯定需要Unsafe(根据内存偏移量 直接从内存中读取修改对象属性）
static final sun.misc.Unsafe U;
```



### 内部类

##### ForwardingNode 转发节点

在扩容时用来标记这个节点已经处理过了。这类节点的hash值为  -1

```java
static final class ForwardingNode<K,V> extends Node<K,V> {
    final Node<K,V>[] nextTable;// 扩容时新table的引用
    ForwardingNode(Node<K,V>[] tab) {
        // 常量MOVED = -1
        super(/*hash*/MOVED, /*k*/null, /*v*/null, /*next*/null);
        this.nextTable = tab;
    }
    ...
}
```



##### TreeNode 红黑树节点

继承自 Map.Node  维护了二叉树节点的属性

```java
static final class TreeNode<K,V> extends Node<K,V> {
    TreeNode<K,V> parent;  // red-black tree links
    TreeNode<K,V> prev;    // needed to unlink next upon deletion
    TreeNode<K,V> left, right;
    boolean red;// 节点颜色

    final TreeNode<K,V> findTreeNode(int h, Object k, Class<?> kc) {
        ...
    }
}
```



##### TreeBin 存储树的容器

封装 TreeNode 的容器，它提供转换黑红树的一些条件和锁的控制

```java
static final class TreeBin<K,V> extends Node<K,V> {
    TreeNode<K,V> root;
    volatile TreeNode<K,V> first;
    volatile Thread waiter;
    volatile int lockState;// 锁状态
    // values for lockState 
    static final int WRITER = 1; // set while holding write lock
    static final int WAITER = 2; // set when waiting for write lock
    static final int READER = 4; // increment value for setting read lock
    
    private final void lockRoot() {
        if (!U.compareAndSwapInt(this, LOCKSTATE, 0, WRITER)) // 如果cas失败就去 竞争锁
            contendedLock(); // offload to separate method
    }
    private final void unlockRoot() {
        lockState = 0;
    }
    
	// 阻塞等待root锁
    private final void contendedLock() {
        boolean waiting = false;
        for (int s;;) {
            if (((s = lockState) & ~WAITER) == 0) {
                if (U.compareAndSwapInt(this, LOCKSTATE, s, WRITER)) {
                    if (waiting)
                        waiter = null;
                    return;
                }
            }
            else if ((s & WAITER) == 0) {
                if (U.compareAndSwapInt(this, LOCKSTATE, s, s | WAITER)) {
                    waiting = true;
                    waiter = Thread.currentThread();
                }
            }
            else if (waiting)
                LockSupport.park(this);// 挂起当前线程
        }
    }

```



### 线程安全机制

synchronized、 CAS

通过一系列对volatile变量的Unsafe操作来保证读到最新的数据

```java
static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
    return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);// JDK源码中好多都是记录该值相对于当前对象的偏移量
}
static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i,
                                    Node<K,V> c, Node<K,V> v) {
    return U.compareAndSwapObject(tab, ((long)i << ASHIFT) + ABASE, c, v);
}
static final <K,V> void setTabAt(Node<K,V>[] tab, int i, Node<K,V> v) {
    U.putObjectVolatile(tab, ((long)i << ASHIFT) + ABASE, v);
}
```



### 功能 & 实现

> 先看看大致步骤吧，不然文字很难描述的那么多分支，直线往下看会有点找不着北。





#### 构造函数

构造函数中没有任何操作，即使是有参数的构造函数，也只是设置`sizeCtl`等一些字段的值。

除非参数中有Map实例，会调用`putAll(map)`。 真正的初始化是在put()方法里。

```java
public ConcurrentHashMap() {
}
public ConcurrentHashMap(int initialCapacity) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException();
    int cap = ((initialCapacity >= (MAXIMUM_CAPACITY >>> 1)) ?
               MAXIMUM_CAPACITY :
               tableSizeFor(initialCapacity + (initialCapacity >>> 1) + 1));
    this.sizeCtl = cap;
}
public ConcurrentHashMap(Map<? extends K, ? extends V> m) {
    this.sizeCtl = DEFAULT_CAPACITY;
    putAll(m);// 只有传入map实例，才会调用put而真正初始化ConcurrentHashMap
}
```



#### 三个原子操作

```java
// 获取tab在i位置上的节点
static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
    return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
}

// CAS更新tab[i]
static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i, Node<K,V> c, Node<K,V> v) {
    return U.compareAndSwapObject(tab, ((long)i << ASHIFT) + ABASE, c, v);
}

// 设置节点位置的值，只在持有锁的时候被调用
static final <K,V> void setTabAt(Node<K,V>[] tab, int i, Node<K,V> v) {
    U.putObjectVolatile(tab, ((long)i << ASHIFT) + ABASE, v);
}
```



#### get(K)

> 使用volatile修饰变量，直接get，不会获取到旧值。
>
> ```java
> public V get(Object key) {
>   Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
>   int h = spread(key.hashCode());//散列 得到key在数组中的位置
>   if ((tab = table) != null && (n = tab.length) > 0 &&
>       (e = tabAt(tab, (n - 1) & h)) != null) {// table不为空，table[i]不为空
>       if ((eh = e.hash) == h) {
>           if ((ek = e.key) == key || (ek != null && key.equals(ek)))
>               return e.val;// table[i]就是要找的 直接返回
>       }
>       else if (eh < 0)// 是转发节点 forwardingNode(正在扩容)，去新table中寻找
>           return (p = e.find(h, key)) != null ? p.val : null;
>       while ((e = e.next) != null) {// 遍历
>           if (e.hash == h &&
>               ((ek = e.key) == key || (ek != null && key.equals(ek))))
>               return e.val;
>       }
>   }
>   return null;
> }
> ```





#### put(K, V)

使用乐观锁cas + synchronized实现并发插入/更新，如果有锁竞争才synchronized锁定`head` / `root`，大致流程：

> 1. 如果还没有初始化：先调用`initTable()`方法来进行初始化；
>
> 2. 如果没有 hash 冲突：直接 CAS 插入；（即table[i]就一个node 或 table[i]还没有node）
>
> 3. 如果正在进行扩容：就协助扩容；
>
> 4. 如果存在 hash 冲突：
>
>    synchronized锁定table[i]处的节点 来保证线程安全，并根据链表当前的结构进行插入
>
>    - 链表：直接遍历到尾端插入；
>    - 红黑树：按照红黑树结构插入；
>
> 5. 如果链表的长度>=8，就再次调用`treeifyBin()`，尝试转换成红黑树；
>
> 6. 最后调用`addCount()`方法统计`size`，并且检查是否需要扩容

```java
public V put(K key, V value) {
    return putVal(key, value, false);
}
static final int spread(int h) {// 其实就是取hash值
    return (h ^ (h >>> 16)) & HASH_BITS;// HASH_BITS = 0x7fffffff
}
/** Implementation for put and putIfAbsent */
final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
    int hash = spread(key.hashCode());
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        if (tab == null || (n = tab.length) == 0)// 1：还未初始化
            tab = initTable();
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {// 2：对应的位置没有node
            if (casTabAt(tab, i, null, new Node<K,V>(hash, key, value, null)))
                break;// no lock when adding to empty bin（添加到空容器中不加锁）
	            // CAS成功才会break，否则接着for循环
        }
        else if ((fh = f.hash) == MOVED)// 3：正在扩容（table[i]这个节点是个forwardingNode）
            tab = helpTransfer(tab, f);// 协助扩容
        else {// 4：存在hash冲突
            V oldVal = null;
            synchronized (f) {// 锁定 table[i]链表的头结点
                if (tabAt(tab, i) == f) {
                    if (fh >= 0) {// hash值正常
                        binCount = 1;// 记录链表有多少个node
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            if (e.hash == hash &&
                                ((ek = e.key) == key || (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                //onlyIfAbsent:只在不存在的时候执行 true:不覆盖旧value; false:覆盖旧value
                                if (!onlyIfAbsent)
                                    e.val = value;// 覆盖value
                                break;// 完事
                            }
                            Node<K,V> pred = e;// 保留当前node的指针，e指向下一个node
                            if ((e = e.next) == null) {// 没有下一个就加到tail
                                pred.next = new Node<K,V>(hash, key, value, null);
                                break;
                            }
                        }
                    }
                    // fh<0（红黑树节点的hash都是-2）
                    else if (f instanceof TreeBin) {// 红黑树结构 
                        Node<K,V> p;
                        binCount = 2;// 最少有两个node
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key, value)) != null) {
                            // put红黑树节点，可能是插入一个节点，可能是替换一个节点的val，这个方法返回对应的节点
                            oldVal = p.val;
                            if (!onlyIfAbsent) // 决定是否覆盖oldValue
                                p.val = value;
                        }
                    }
                }
            }
            // 最后计算数量
            if (binCount != 0) {
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);// 节点数>=8 申请转化为红黑树
                if (oldVal != null)
                    return oldVal; // 返回非空oldValue
                break;
            }
        }
    }
    addCount(1L, binCount);// 统计size
    return null;
}
```



#### initTable() 初始化table

```java
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    while ((tab = table) == null || tab.length == 0) {
        if ((sc = sizeCtl) < 0)// 说明有其他线程正在扩容
            Thread.yield(); // lost initialization race; just spin   自旋等待初始化完成。
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {// cas设置sc为-1，表示正在初始化
            try {
                if ((tab = table) == null || tab.length == 0) {
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;// sc为正数标识阈值
                    @SuppressWarnings("unchecked")
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    sc = n - (n >>> 2);// 并设置阈值。size达到sc就扩容。
                    // >>>2就是/4,  n-n/4就是n=n*0.75呗
                }
            } finally {
                sizeCtl = sc;// 更新sizeCtl属性
            }
            break;
        }
    }
    return tab;
}
```



#### helpTransfer() 协助扩容

```java
final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
    Node<K,V>[] nextTab; int sc;
    if (tab != null && (f instanceof ForwardingNode) && // f是正在转化的node
        (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {// 并且 newTable已经存在
        int rs = resizeStamp(tab.length);
        while (nextTab == nextTable && table == tab && // 验证newTable和oldTable
               (sc = sizeCtl) < 0) {// 并且sc是正在扩容
            if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                sc == rs + MAX_RESIZERS || transferIndex <= 0)
                break;
            if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {
                transfer(tab, nextTab);// 调用扩容方法
                break;
            }
        }
        return nextTab;
    }
    return table;
}
```



#### transfer() 扩容

##### 步骤

> 1. 开辟新table空间：`nextTable`
>
> 2. 遍历table，计算当前线程的步长(拿到自己的子任务)  ----- 每个线程都会计算自己能拿到的任务，以此方式协作！
>
> 3. 执行自己的任务，从后往前遍历（table[15]->table[0]）
>
>    > 比如16扩容到32，就是从15遍历到0，不会有其他线程再参与扩容了(因为最小步长16)
>
>    对table[i]的 链表 / 红黑树 进行遍历
>
>    - 链表（两次for遍历）
>
>      > 1. 找到链表尾端(newindex都相同的)最长子链表(图1的678)，并设置下次遍历的终点`lastRun`为这个子链表的head(图1的6)
>      >
>      > 2. 再次遍历table[i]，for [head,  `lastRun`) （lastRun就比如图1的6）
>      >
>      >    > 使用**头插法**把node都插入到新low链表/新high链表的前边。
>      >
>      >    结果会是有一个新链表的顺序会倒置（不是第一次for拿出的子链表）（图1的结果：5123 和 4678）
>
>    - 红黑树
>
> 4. 把新链表放到新table对应的索引位

##### 源码

```java
// Moves and/or copies the nodes in each bin to new table. 将所有node移动到新table中
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
    // ---------------- 1：给当前线程分配任务
    int n = tab.length, stride;// stride：步长，每个线程负责table中一步之内的所有链表
    if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
        stride = MIN_TRANSFER_STRIDE; // subdivide range 细分范围  
    	// 最少16，也就是从16扩容到32的时候只会有一个线程参与扩容
    if (nextTab == null) {            // initiating
        try {
            @SuppressWarnings("unchecked")
            Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];// newTable容量为原来的两倍
            nextTab = nt;
        } catch (Throwable ex) {      // try to cope with OOME ..尝试处理OOM
            sizeCtl = Integer.MAX_VALUE;
            return;
        }
        nextTable = nextTab;
        transferIndex = n;// 当前线程的扩容任务是从 n开始
    }
    // ---------------- 2：执行当前线程的扩容子任务（遍历当前线程负责的一步范围内所有的链表）
    int nextn = nextTab.length;
    ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);// 创建forward节点
    boolean advance = true;
    boolean finishing = false; // to ensure sweep before committing nextTab 确保在提交nextTab之前扫描
    // i 指当前处理的槽位序号， bound 指需要处理的槽位边界，先处理槽位15的节点
    for (int i = 0, bound = 0;;) {// 遍历当前线程负责的一步范围内所有的链表
        Node<K,V> f; int fh;
        // ---------------- 2.1：自旋设置transferIndex的值 并初始化i 和 bound
        while (advance) {
            int nextIndex, nextBound;
            if (--i >= bound || finishing)
                advance = false;
            else if ((nextIndex = transferIndex) <= 0) {
                i = -1;
                advance = false;
            }
            // cas设置transferIndex属性的值
            else if (U.compareAndSwapInt (this, TRANSFERINDEX, nextIndex,
                      nextBound = (nextIndex > stride ? nextIndex - stride : 0))
                    ) {
                bound = nextBound;
                i = nextIndex - 1;// 初始化 i 和 bound
                advance = false;
            }
        }
        // ----------------↓↓ todo ??
        if (i < 0 || i >= n || i + n >= nextn) {// todo ??
            int sc;
            if (finishing) {// 如果已经复制完所有节点
                nextTable = null;
                table = nextTab;// table变量 指向新table
                sizeCtl = (n << 1) - (n >>> 1);// sc= n*2 - n/2 相当于 sc=n*1.5，阈值变为oldCap的1.5倍
                return;
            }
            if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {// cas替换sc的值，通知其他线程 多了一个线程参与扩容
                if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                    return;
                finishing = advance = true;
                i = n; // recheck before commit
            }
        }
        // ----------------↑↑ todo ??
        else if ((f = tabAt(tab, i)) == null)// 如果节点为null，通过cas将forward节点放到该位置
            advance = casTabAt(tab, i, null, fwd);
        else if ((fh = f.hash) == MOVED) // 如果该节点已经是forward节点
            advance = true; // already processed
        else { // ---------------- 2.2：执行任务，把table[i]这个链表 转移到nextTable
            synchronized (f) {// 对节点加锁
                if (tabAt(tab, i) == f) {
                    Node<K,V> ln, hn;// newIndex两种可能：oldIndex 或 oldIndex+oldCap
                    // ---------------- 2.2.0：链表
                    if (fh >= 0) {
                        //n:oldCap，unBit是否为0 表示这个节点是在low还是high
                        int runBit = fh & n;
                        Node<K,V> lastRun = f;// 记录最后需要处理的节点
                        // ---------------- 2.2.1：遍历链表
                        for (Node<K,V> p = f.next; p != null; p = p.next) {
                            int b = p.hash & n;// mask之后是否为0
                            if (b != runBit) {// 如果节点p和节点f 的newIndex不同 如图1，p是节点4时才会第一次进入这个if
                                runBit = b;// 更新runBit
                                lastRun = p;// 更新要处理的节点
                            }
                        }
                        // 这次for结束后，lastRun及其后所有节点newIndex均一致，如'图1'中的678，6是lastRun
                        if (runBit == 0) {// 将lastRun截下来放到新table中对应的槽
                            ln = lastRun;
                            hn = null;
                        }
                        else {
                            hn = lastRun;
                            ln = null;
                        }
                        // ---------------- 2.2.1：再次遍历链表，范围：[head, lastRun)
                        for (Node<K,V> p = f; p != lastRun; p = p.next) {
                            int ph = p.hash; K pk = p.key; V pv = p.val;
                            // 正向遍历，头插法，所以另一个newIndex的链表(图1中的蓝色)节点会倒置
                            if ((ph & n) == 0)
                                ln = new Node<K,V>(ph, pk, pv, /*next*/ln);// 构造函数第四个参数是next，懂了吧。头插法
                            else
                                hn = new Node<K,V>(ph, pk, pv, hn);
                        }
                        // ---------------- 2.2.2：放到newTab中对应的index，完活
                        setTabAt(nextTab, i, ln);
                        setTabAt(nextTab, i + n, hn);
                        setTabAt(tab, i, fwd);// 在table i 位置处插上ForwardingNode 表示该节点已经处理过了
                        advance = true;
                    }
                    // ---------------- 2.2.0：红黑树 
                    else if (f instanceof TreeBin) {
                        TreeBin<K,V> t = (TreeBin<K,V>)f;
                        TreeNode<K,V> lo = null, loTail = null;// newIndex为low的槽 对应的root和tail
                        TreeNode<K,V> hi = null, hiTail = null;// newIndex为high的槽 对应的root和tail
                        int lc = 0, hc = 0;// 记录新table中 low槽和high槽的节点数，后边会检查是否需要转为链表结构
                        for (Node<K,V> e = t.first; e != null; e = e.next) {
                            int h = e.hash;
                            TreeNode<K,V> p = new TreeNode<K,V>
                                (h, e.key, e.val, /*next*/null, /*parent*/null);
                            if ((h & n) == 0) {// low槽
                                if ((p.prev = loTail) == null)
                                    lo = p;
                                else
                                    loTail.next = p;
                                loTail = p;
                                ++lc;
                            }
                            else {// high槽
                                if ((p.prev = hiTail) == null)
                                    hi = p;
                                else
                                    hiTail.next = p;
                                hiTail = p;
                                ++hc;
                            }
                        }
                        // 如果扩容后节点数<6 转为链表
                        ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                        (hc != 0) ? new TreeBin<K,V>(lo) : t;
                        hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                        (lc != 0) ? new TreeBin<K,V>(hi) : t;
                        // 放到新table对应的槽，完活
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

> 图1：
>
> <img alt="节点newIndex不同的链表，图在github" src="https://raw.githubusercontent.com/melopoz/pics/master/img/%E8%8A%82%E7%82%B9newIndex%E4%B8%8D%E5%90%8C%E7%9A%84%E9%93%BE%E8%A1%A8.png" style="zoom:50%;" />
>
> 这个链表最后会变成  `5->3->2->1` 和 `4->6->7->8`



#### addCount()  统计size

```java
@sun.misc.Contended static final class CounterCell {
    volatile long value;
    CounterCell(long x) { value = x; }
}

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

@sun.misc.Contended static final class CounterCell {
    volatile long value;
    CounterCell(long x) { value = x; }
}

private final void addCount(long x, int check) {
    CounterCell[] as; long b, s;
    // 初始化时counterCells为空，在并发量很高时，如果存在两个线程同时执行CAS修改baseCount值，
    // 则失败的线程会继续执行方法体中的逻辑，使用CounterCell记录元素个数的变化
    if ((as = counterCells) != null ||
        !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
        // 使用CounterCell记录元素个数的变化
        CounterCell a; long v; int m;
        boolean uncontended = true;
        // 如果CounterCell数组counterCells为空，调用fullAddCount()方法进行初始化，
        // 并插入对应的记录数，通过CAS设置cellsBusy字段，只有设置成功的线程才能初始化CounterCell数组
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

todo addCount       T.T 

图1来自  https://www.jianshu.com/p/0e435f39e796  ，正好看看dalao对addCount怎么说的，还有红黑树



todo 红黑树 https://blog.csdn.net/v_july_v/article/details/6105630





## 总结

### 1.7和1.8区别（1.8的优化）

1. 1.8 的结构和HashMap大致相同（1.7的ConcurrentHashMap其实就和1.8的HashMap差不多了）；

2. 1.8 取消segment分段锁，不用ReentrantLock了，而是使用很多乐观锁-自旋CAS，

   乐观失败就用synchronized锁定table[i]的链表的head。锁粒度更细，并发性高；

   > 锁粒度降低，synchronized并不比ReentrantLock差了。
   >
   > 本来ReentrantLock的Condition可以在粗粒度的场景中提高性能（我觉得算是变相降低锁粒度吧）。
   >
   > 锁粒度降下来之后，ReentrantLock就没必要了。
   >
   > > 而且synchronized可以基于jvm优化，使用关键字比使用api确实更加自然。
   >
   > > 数据量很大的话，使用基于 api 的 ReentrantLock 会比 synchronized 占用更多的内存

3. 可以协助扩容，扩容的时候1.8用到了头插法，部分node顺序会倒置；

4. 高效更新元素个数，类似LongAdder 。。。？todo



### HashMap和ConcurrentHashMap的区别

1. HashMap线程不安全，..；
2. ConcurrentHashMap扩容的时候会先拿当前线程的子任务（步长 `stride`），HashMap没这么多事；
3. ConcurrentHashMap扩容的时候有用到头插法，部分node的顺序会倒置，而HashMap用的尾插法；
4. 1.7的ConcurrentHashMap的put和get操作都要两次hash