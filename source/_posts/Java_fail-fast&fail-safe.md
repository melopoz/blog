---
title: fail-fast & happens-before
tags: 
categories:
  - [Java]
---



https://blog.csdn.net/Kato_op/article/details/80356618

## fail-fast

在使用迭代器遍历一个集合的时候，例如foreach(实现还是iterator)，如果遍历过程中修改了集合对象的内容，会抛出ConcurrentModificationException异常。

> 补充： 实现Iterator接口就能进行迭代，外部写foreach也是用的这个iter，java都会编译成使用iter进行迭代的代码

原理：

1. 迭代器在遍历时直接访问集合中的内容,并且在遍历过程中使用一个modCount变量,
2. 集合中在被遍历期间如果内容发生变化,就会改变modCount的值,
3. 每当迭代器使用 hashNext()/next()遍历下一个元素之前,都会检测modCount变量和expectedmodCount值是否相等,
4. 如果相等就返回遍历,否则抛出异常,终止遍历.



## fail-safe

JUC下的包采用安全失败机制，遍历时的修改操作会在原集合的副本中进行modify，然后再把副本的引用给原集合。

##### 注意！

> 在遍历过程中添加的元素是不会在本次遍历过程中遍历到的！
>
> 因为遍历的是开始时的副本。

##### demo

```java
// list中有10个元素
ArrayList<Integer> list = new ArrayList<>();
for (int i = 0; i < 10; i++) list.add(i);

// li中有100个元素，遍历时如果list中也有就添加到li中，最终li会有110个
CopyOnWriteArrayList<Integer> li = new CopyOnWriteArrayList<>();
for (int i = 0; i < 100; i++) li.add(i);

int count = 0;
for (Integer i : li) {
    System.out.println(count++ + "\t" + i);
    if(list.contains(i)){
        li.add(i);
    }
}
System.out.println(li.size());
System.out.println(Arrays.toString(li.toArray()));
```

打印结果：

```
start
0	0 // 
1	1
2	2
3	3
4	4
5	5 // 这些值都是被改掉的，没有遍历出来新结果5000
6	6
7	7
8	8
9	9 // 
10	10
11	11
12	12
13	13
14	14
15	15
16	16
17	17
18	18
19	19 // 后边新添加的10个元素也没有遍历到
30
[0, 1000, 2000, 3000, 4000, 5000, 6000, 7000, 8000, 9000, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
```



## happens-before

。。todo



大概就是     读之前的写操作不能放到读之后    lock unlock read write load store assign use  关乎这些指令的重排序