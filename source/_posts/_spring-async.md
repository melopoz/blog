title: spring中异步执行的实现方式
tags: async
categories: java
date: 2021/06/27 13:00
updated: 2021/06/27 13:00

---



#### @Async方法

1. 需要使用@EnableAsync启动异步注解
2. 返回值必须是void/Future<T\>
3. @Async注解有一个字段：value，含义是Spring容器中的线程池Bean，不指定则使用默认的线程池
   - 详见detrermineAsyncExecutor(Method method)方法。只有这一处代码使用了Async.value
   - 默认线程池是 core=8，queue=Int.max。推荐使用自己的配置的线程池。



#### ComplateFuture

