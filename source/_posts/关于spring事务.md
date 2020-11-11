https://my.oschina.net/u/4579410/blog/4525713

看完这个明白的



想想aop，

犯的错：

> 在一个serviceA的非事务方法中调用 this.func2()   func2是个事务方法，
>
> func2中调用了 serviceB的func1()  serviceB.func1() 是事务方法

> 结果：    在serviceB.func1()最后抛异常， serviceB回滚了  serviceA.func2()的数据落库了。。

 

原理 ：

在serviceA的代理对象的调用过程中，serviceA的方法内部调用了本service的方法，aop只在 最外层的方法（serviceA.func1）进行了增强，也就是 事务开启和回滚都应该加在这里，但是serviceA.func1 没有@Transaction。。。所以 serviceA.func2() 的声明式事务并没有生效。



