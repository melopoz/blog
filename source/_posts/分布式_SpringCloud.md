---
title: SpringCloud
tags: 
  - 分布式
categories:
  - [分布式]
date: 2021/01/01 20:46:25
updated: 2021/01/01 20:46:25
---



## 微服务架构要处理的四个问题：

1. api
2. 通信：http、RPC
3. 服务注册和发现（解决高可用问题）
4. 熔断机制

## CAP原则：

- C	consistency	强一致性
- A     availability      可用性
- P      partition tolerance      分区容错性  （分区相当于对通信的时限要求）

只能满足其中两点。P：当系统不能在时限内完成数据一致性，就需要在C和A之间做出选择。

zookeeper：CP

Eureka：AP

> 所以Eureka的自我保护机制可以很好地应对网络故障导致很多节点失去联系的情况，不会像zookeeper那样整个集群都不可用了。

## 目前的三个解决方案：

1. Apache Dubbo Zookeeper

   api：需要整合第三方 或者自行实现

   通信：dubbo通信框架可以使用RPC或HTTP

   服务注册：zookeeper

   熔断机制：无

2. Spring Cloud NetFlix

   api：网关，zuul组件

   通信：Feign  HttpClient 	http通信方式，同步，阻塞

   服务注册：Eureka

   熔断机制：Hystrix

   负载均衡：Ribbon

3. Spring Cloud Alibaba

   2019.9开始使用 一站式解决方案



Dubbo是指RPC框架，Spring Cloud是微服务架构的一站式解决方案。本质不同。

# Spring Cloud

使用restful的http通信

### 项目结构

- 后端api项目(provider)：使用controller提供api。

- 前端consumer项目：使用RestTemplate发送restful请求，调用api项目提供的服务。

- 使用Eureka当做服务注册中心。



### Eureka 服务注册中心

##### 两个组件：

>  Eureka Server 和 Eureka Client

- server提供服务注册服务，就像zookeeper

-  client是一个java客户端，有一个内置复杂均衡器(轮询)。 应用启动之后想server发送心跳，server在多个心跳周期中没有收到服务提供者的心跳，就移除该provider。

##### 三个角色：

> Eureka Server  Eureka Provider  Eureka Consumer: 

##### 一个EurekaServer项目

依赖

```xml
	<dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-eureka-server</artifactId>
            <version>1.4.4.RELEASE</version>
        </dependency>
    </dependencies>
```

启动类

```java
@SpringBootApplication
@EnableEurekaServer
public class EurekaServer {
    public static void main(String[] args) {
        SpringApplication.run(EurekaServer.class, args);
    }
}

```

application.yml

```yaml
server:
  port: 9101

spring:
  application:
    name: eureka-server

eureka:
  instance:
    hostname: eureka9101.com
  client:
    register-with-eureka: false #不登记 表示自己是注册中心
    fetch-registry: false #不抓取注册中心 自己作为注册中心
    service-url:
      #单机
      #defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka #注册服务的地址
      #集群 将其他的eureka注册中心写进来
      defaultZone: http://eureka9102.com:9102/eureka,http://eureka9103.com:9103/eureka
```

##### 一个EurekaClient-服务提供者项目(后端api)

> ​	springboot项目结构

依赖

```xml
<!--boot web starter 版本控制在父pom的dependenciesManager-->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>		
<!--eureka-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-eureka</artifactId>
    <version>1.4.4.RELEASE</version>
</dependency>
<!--actuator-->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
    <version>2.1.14.RELEASE</version>
</dependency>
```

启动类

```java
@SpringBootApplication
//...其他注解
@EnableEurekaClient //eureka client
@EnableDiscoveryClient //完善注册中心的信息
public class ApiApplication {
    public static void main(String[] args) {
        SpringApplication.run(ApiApplication.class, args);
    }
}
```

application.yml

```yaml
server:
  port: 9001
spring:
  application:
    name: playerapi
eureka:
  client:
    service-url:
      defaultZone: http://eureka9101.com:9100/eureka,http://eureka9102.com:9102/eureka,http://eureka9103.com:9103/eureka
  instance:
    instance-id: api

#可以在eureka中显示的服务实例的actuator/info中看到 需要spring-boot-starter-actuator依赖
info:
  api.name: firstspringcloud-api
  api.msg: this is api
```



### Feign 使用接口方式调用服务，类似Dubbo

前端app调用后端api服务的方式：

- 使用RestTemplate

  > ##### 一个前端(服务消费者)项目(boot web) consumer 使用RestTemplate通信
  >
  > 依赖
  >
  > ```xml
  > <dependencies>
  >     <dependency>
  >         <groupId>com.melopoz</groupId>
  >         <artifactId>commons</artifactId>
  >         <version>1.0-SNAPSHOT</version>
  >     </dependency>
  >     <dependency>
  >         <groupId>org.springframework.boot</groupId>
  >         <artifactId>spring-boot-starter-web</artifactId>
  >     </dependency>
  >     <!--Ribbon-->
  >     <dependency>
  >         <groupId>org.springframework.cloud</groupId>
  >         <artifactId>spring-cloud-starter-ribbon</artifactId>
  >         <version>1.4.4.RELEASE</version>
  >     </dependency>
  >     <!--eureka-->
  >     <dependency>
  >         <groupId>org.springframework.cloud</groupId>
  >         <artifactId>spring-cloud-starter-eureka</artifactId>
  >         <version>1.4.4.RELEASE</version>
  >     </dependency>
  > </dependencies>
  > ```
  >
  > 启动类
  >
  > ```java
  > @SpringBootApplication
  > @EnableEurekaClient
  > public class FrontApplication1 {
  >     public static void main(String[] args) {
  >         SpringApplication.run(FrontApplication1.class, args);
  >     }
  > }
  > 
  > ```
  >
  > config
  >
  > ```java
  > @Configuration
  > public class Beans {
  >     @Bean
  >     @LoadBalanced//负载均衡的restTemplate
  >     RestTemplate restTemplate() {
  >         return new RestTemplate();
  >     }
  > }
  > ```
  >
  > controller
  >
  > ```java
  > 	// PLAYERAPI 是Eureka中的服务的Application的名称  
  > 	private static final String baseUrl = "http://PLAYERAPI/player";
  > 
  >     @Autowired
  >     private RestTemplate restTemplate;
  > 
  >     @GetMapping("/list")
  >     public List list() {
  >         return restTemplate.getForObject(baseUrl + "/list", List.class);
  >     }
  > ```
  >
  > application.yml
  >
  > ```yaml
  > server:
  >   port: 10001
  > eureka:
  >   client:
  >     register-with-eureka: false #不向eureka注册自己
  >     service-url:
  >       defaultZone: http://eureka9101.com:9101/eureka,http://eureka9102.com:9102/eureka,http://eureka9103.com:9103/eureka
  > ```
  >
  > 

- 在commons中做公共service包，使用Feign进行服务注册

  > commons项目 依赖
  >
  > ```xml
  > <!--feign-->
  > <dependency>
  >     <groupId>org.springframework.cloud</groupId>
  >     <artifactId>spring-cloud-starter-feign</artifactId>
  >     <version>1.4.4.RELEASE</version>
  > </dependency> 
  > ```
  >
  > commons项目 service包
  >
  > ```java
  > @Component
  > @FeignClient(value = "PLAYERAPI", fallbackFactory = FrontPlayerServiceFallbackFactory.class)
  > public interface FrontPlayerService {
  >     @GetMapping("/player/list")
  >     List<Player> list();
  >     @GetMapping("/player/{id}")//在Service中必须有value=，不然报错
  >     Player getById(@PathVariable(value = "id") Long id);
  >     @GetMapping("/player/test")
  >     Player test();
  > }
  > ```
  >
  > feignFront项目 依赖
  >
  > ```xml
  > <dependencies>
  >     <dependency>
  >         <groupId>com.melopoz</groupId>
  >         <artifactId>commons</artifactId>
  >         <version>1.0-SNAPSHOT</version>
  >     </dependency>
  >     <dependency>
  >         <groupId>org.springframework.boot</groupId>
  >         <artifactId>spring-boot-starter-web</artifactId>
  >     </dependency>
  >     <!--eureka-->
  >     <dependency>
  >         <groupId>org.springframework.cloud</groupId>
  >         <artifactId>spring-cloud-starter-eureka</artifactId>
  >         <version>1.4.4.RELEASE</version>
  >     </dependency>
  >     <!--feign 可以没有 commons已经引入-->
  >     <!-- <dependency>
  >         <groupId>org.springframework.cloud</groupId>
  >         <artifactId>spring-cloud-starter-feign</artifactId>
  >         <version>1.4.4.RELEASE</version>
  >     </dependency> -->
  > </dependencies>
  > ```
  >
  > controller
  >
  > ```java
  > @RestController
  > @RequestMapping("/player")
  > public class PlayerController {
  >     @Autowired
  >     private FrontPlayerService frontPlayerService; //直接注入commons项目中的service
  >     @GetMapping("/list")
  >     public List list() {
  >         return frontPlayerService.list();
  >     }
  >     @GetMapping("/{id}")//@PathVariable在Service接口中必须有value=
  >     public Player getById(@PathVariable Long id) {
  >         return frontPlayerService.getById(id);
  >     }
  >     @GetMapping("/test")
  >     public Player test() {
  >         return frontPlayerService.test();
  >     }
  > }
  > ```



### Ribbon 负载均衡

> 在前端项目(consumer)实现负载均衡

```java
@Configuration
public class Beans {
    @Bean
    @LoadBalanced//负载均衡的restTemplate
    RestTemplate restTemplate() {
        return new RestTemplate();
    }
}
```

轮询 随机 权重。。也可以自定义实现负载均衡算法



### Hystrix 服务熔断 服务降级

> - 服务熔断：如果后端api中的某个服务在相应的时候超时、抛异常了。就返回另一个备用方法。是为了安全
> - 服务降级：如果某个时间段要调用太多A服务，几乎不调用B服务。就直接使用B服务备用方法的返回值，不让b服务真正运行，从而减小服务器的负载，让服务器更有性能去运行a服务。是为了效率
>
> 备用方法：返回当前服务不可用的信息。

后端实现服务熔断的三种方法：

1. 超时熔断
2. 线程池隔离
3. 信号量隔离

##### demo:

- 使用RestTemplate调用，并在后端模拟服务故障，实现**服务熔断**

  > 在后端提供api服务的项目的controller层的接口方法中如果有异常，就调用熔断的方法(返回null格式的数据)
  >
  > 启动类
  >
  > ```java
  > @EnableCircuitBreaker//hystix  开启熔断机制
  > ```
  >
  > controller
  >
  > ```java
  > xxxController {
  >    	// 模拟服务调用异常，熔断
  >     @HystrixCommand(fallbackMethod = "hystrixGetById") // 熔断后调用的方法
  >     @GetMapping("/{id}")
  >     public Player getById(@PathVariable Long id) {
  >         Player player = playerService.getById(id);
  >         if (player == null) {
  >             System.out.println("throw exception...");
  >             throw new RuntimeException("不存在id为" + id + "的Player数据");
  >         }
  >         return player;
  >     }
  >     public Player hystrixGetById(@PathVariable Long id) {
  >         System.out.println("hystrixGetById()...");
  >         return new Player().setId(id);
  >     }
  >     // ...
  >     //模拟服务调用超时，降级
  > 	//超时降级配置
  > 	@HystrixCommand(
  >       commandKey = "list",
  >       commandProperties = {
  >        @HystrixProperty(name = "execution.timeout.enabled",value = "true"),
  >        @HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds",
  >                     	value = "2000"),
  >       },
  >       fallbackMethod = "listFallbackMethod4Timeout"
  >     )
  >     @GetMapping("/list")
  >     public List<Player> list() {
  >         try {
  >             int ms = new Random().nextInt(5);
  >             System.out.println("休眠" + ms + "s");
  >             Thread.sleep(ms * 1000);
  >         } catch (InterruptedException e) {
  >             e.printStackTrace();
  >         }
  >         return playerService.list();
  >     }
  >     public List<Player> listFallbackMethod4Timeout() {
  >         System.out.println("listFallbackMethod4Timeout()...list调用超时了");
  >         return new ArrayList<Player>();
  >     }
  > }
  > ```

- 使用feign接口方式调用服务，并在前端实现**服务降级**

  > 使用Feign的接口方式调用的时候，在service中加入FallbackFactory，create()方法返回新的service接口的实例，实现服务降级
  >
  > 在commons项目中
  >
  > service.xxxService
  >
  > ```java
  > @Component
  > @FeignClient(value = "PLAYERAPI", 
  >              // 加入fallback处理
  >              fallbackFactory = FrontPlayerServiceFallbackFactory.class)
  > public interface FrontPlayerService {
  > 
  >     @GetMapping("/player/list")
  >     List<Player> list();
  > 
  >     @GetMapping("/player/{id}")//在Service中必须有value=，不然报错
  >     Player getById(@PathVariable(value = "id") Long id);
  > 
  >     @GetMapping("/player/test")
  >     Player test();
  > }
  > ```
  >
  > config.fallbackFactory
  >
  > ```java
  > @Component
  > public class FrontPlayerServiceFallbackFactory implements FallbackFactory {
  >     public FrontPlayerService create(Throwable throwable) {
  >         return new FrontPlayerService() {
  >             public List<Player> list() {
  >                 System.out.println("服务降级...list()...");
  >                 return new ArrayList<Player>();
  >             }
  > 
  >             public Player getById(Long id) {
  >                 System.out.println("服务降级...getById()...");
  >                 return new Player().setId(id);
  >             }
  > 
  >             public Player test() {
  >                 System.out.println("服务降级...test()...");
  >                 return new Player();
  >             }
  >         };
  >     }
  > }
  > ```
  >
  > application.yml
  >
  > ```yml
  > feign:
  >   hystrix:
  >     enabled: true #开启服务降级
  > ```

### Dashboard 监控

1. 导入依赖

   ```xml
   <!--consumer项目依赖加-->
   <!--hystrix dashboard-->
   <dependency>
       <groupId>org.springframework.cloud</groupId>
       <artifactId>spring-cloud-starter-hystrix-dashboard</artifactId>
       <version>1.4.4.RELEASE</version>
   </dependency>
   ```

   provider项目需要有

   ```xml
   <!--actuator-->
   <dependency>
       <groupId>org.springframework.boot</groupId>
       <artifactId>spring-boot-starter-actuator</artifactId>
       <version>2.1.14.RELEASE</version>
   </dependency>
   ```

   给要被监控的api项目的启动类添加servletBean

   **并且只能对有熔断机制的api进行监控**

   ```java
   @SpringBootApplication
   @EnableTransactionManagement
   @MapperScan("com.melopoz.mapper")
   @EnableEurekaClient
   @EnableDiscoveryClient
   @EnableCircuitBreaker//hystix  开启熔断机制
   public class HystrixApiApplication {
       public static void main(String[] args) {
           SpringApplication.run(HystrixApiApplication.class, args);
       }
   
       @Bean
       public ServletRegistrationBean hystrixMetricsStreamServlet() {
           ServletRegistrationBean registrationBean = new ServletRegistrationBean(new HystrixMetricsStreamServlet());
           registrationBean.addUrlMappings("/hystrix.stream");
           return registrationBean;
       }
   }
   
   ```

   

2. 创建启动类

   ```java
   @SpringBootApplication
   @EnableHystrixDashboard
   public class DashBordApplication {
       public static void main(String[] args) {
           SpringApplication.run(DashBordApplication.class, args);
       }
   }
   ```

3. application.yml

   ```yaml
   server: 
     port: 9999
   ```

   访问http://localhost:9999/hystrix



### Gateway 网关

分发请求到提供服务的api

1. 依赖

   ```xml
   <!--dashboard依赖 加-->
   <!--zuul-->
   <dependency>
       <groupId>org.springframework.cloud</groupId>
       <artifactId>spring-cloud-starter-zuul</artifactId>
       <version>1.4.4.RELEASE</version>
   </dependency>
   ```

2. 启动类

   ```java
   @SpringBootApplication
   @EnableZuulProxy //使用zuul网关
   public class ZuulGatewayApplication {
       public static void main(String[] args) {
           SpringApplication.run(ZuulGatewayApplication.class, args);
       }
   }
   
   ```

3. application.yml

   ```yaml
   server:
     port: 9090
   
   spring:
     application:
       name: zuul-gateway
   
   eureka:
     client:
       service-url:
         defaultZone: http://eureka9101.com:9101/eureka,http://eureka9102.com:9102/eureka,http://eureka9103.com:9103/eureka
     instance:
       instance-id: zuul-gateway
       prefer-ip-address: true # 显示服务的ip
   
   zuul:
     routes: # 一个map 自定义
       playerApi.serverId: player
       playerApi.path: /player/**
     ignored-services: playerapi #隐藏 不可用这个访问
     prefix: /mp
   
   info:
     name: firstspringcloud-zuul gateway
     msg: this is zuul-gateway
   
   ```



### Spring Cloud Config

使用git统一配置项目的config文件。