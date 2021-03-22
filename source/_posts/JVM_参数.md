---
title: JVM的一些参数
tags: 
categories:
  - [Java,JVM]
---





查看jvm参数(参数默认值): ` java -XX:+PrintFlagsFinal -version `

查看java进程的参数的配置: `jinfo -flags [pid]` `

查看java进程的指定参数的配置: `jinfo -flags [option] [pid]`

> jinfo -flags NewRatio 15222





# SoftRefLRUPolicyMSPerMB

> 关系到 软引用 再GC发生时,是否被GC回收
>
> -XX:SoftRefLRUPolicyMSPerMB

> LRU:  Least Recently Used, 最近最少使用.

https://blog.csdn.net/u010833547/article/details/90289325

软引用是在内存空间不足的时候才会被回收, 这只是比较简单的形容. 



首先看一下SoftReference类和描述

```java
public class SoftReference<T> extends Reference<T> {
    /**
     * Timestamp clock, updated by the garbage collector
     */
    static private long clock;// ------------每次GC会更新这个时间戳

    /**
     * Timestamp updated by each invocation of the get method.  The VM may use
     * this field when selecting soft references to be cleared, but it is not
     * required to do so.
     */
    private long timestamp;// ------------每次使用(构造、get)会更新这个时间戳

    public SoftReference(T referent) {
        super(referent);
        this.timestamp = clock;
    }

    public SoftReference(T referent, ReferenceQueue<? super T> q) {
        super(referent, q);
        this.timestamp = clock;
    }

    public T get() {
        T o = super.get();
        if (o != null && this.timestamp != clock)
            this.timestamp = clock;
        return o;
    }

}

```



GC时,回收软引用对象要走LRU算法

> if (       `clock - timestamp <= freespace * SoftRefLRUPolicyMSPerMB`       ) {
>
> ​		不回收这个对象
>
> }

clock 记录的是上一次GC的时间戳, timestamp 记录最近使用这个引用对象的时间, freespace是 剩余可用空间, 

`clock - timestamp` 表示这个引用已经闲置了多久, 

`freespace * SoftRefLRUPolicyMSPerMB` 表示JVM.GC对软引用的忍耐程度. SoftRefLRUPolicyMSPerMB越小, 就会越早回收soft-ref对象. 



如果`clock - timestamp`为负数, 是 本次GC之前,上次GC之后被使用过, 那么本次GC不回收这个soft-ref. 

所以软引用肯定不会被它所经历的第一次GC(本次GC)回收. 

> 只有发生gc时才会计算这个 `clock - timestamp` 的啊.. 
>
> 个人认为必须要 先确定是否回收这个soft-ref, 如果不必回收这个soft-ref, 再更新clock. 
>
> 这样这个clock才能表示 上次GC的时间, 否则这个clock就是当前GC的时间(因为要STW, 所以clock就是当前时间)

SoftRefLRUPolicyMSPerMB的默认值为1000, 也就是1s



### -xx:SoftRefLRUPolicyMSPerMB 的坑

这个值越小, jvm对软引用的忍耐程度就越小, 但是不能设置为0

###### 如果为0: 

每次GC的时候很多软引用

参考: https://blog.csdn.net/qiang_zi_/article/details/100700784

就是jvm利用反射生成的很多软引用对象, 每次GC的时候因为对软引用的忍耐程序为0, 就都给清除了,  清除之后还要用到这些soft-ref, 就又生成, 这样不但没有真正节省出内存空间, 还降低了性能. 

还如上方链接中举例, 可能会造成metaspace不稳定, 导致FULL GC频率飙升



所以不便在一开始就设置SoftRefLRUPolicyMSPerMB, 可以在后期根据情况进行调整.

有可能需要调高SOftRefLRUPolicyMSPerMB, 让一些经常用到的soft-ref对象不会总被回收再创建, 使metaspace空间的使用率更平稳一些.