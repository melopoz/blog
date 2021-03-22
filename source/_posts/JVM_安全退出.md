---
title: JVM安全退出
tags: 
categories:
  - [Java,JVM]
---



## Shutdown Hook

System.getRuntime().addShutdownHook(Thread thread);

```java
public class JVMShutdownHook {
    public static void main(String[] args) throws InterruptedException {
        //hook
        Runtime.getRuntime().addShutdownHook(new Thread(()->{
            System.out.println("jvm要关闭了");
            try {
                TimeUnit.SECONDS.sleep(3);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("拜拜");
        }));

        for (int i = 0; i < 5; i++) {
            System.out.println(i);
            if (i == 3){
                System.exit(0);
            }
        }
    }
}
```

控制台打印：

```
D:\Java\jdk1.8.0_231\bin\java.exe ...
0
1
2
3
jvm要关闭了
//等3s
拜拜

Process finished with exit code 0
```



## java结束程序的方法：

<img src="https://raw.githubusercontent.com/melopoz/pics/master/img/jvm%E5%85%B3%E9%97%AD.png" style="zoom:50%;" />

- System.exit(int status);

  > status表示程序退出的状态码，如果其他程序调用这个程序，这个状态吗就会返回给调用者。
  >
  > 0表示正常退出，非0表示异常终止。