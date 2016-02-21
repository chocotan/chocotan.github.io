---
title: Spring Cloud学习笔记——Netflix的那些东西
date: 2016-02-21 23:04:08
tags: [spring,java]
categories: java
---

最近*微服务*好像很火的样子，我司是用阿里的dubbo来搭建微服务的，最初感觉好牛叉的样子，但入职一年接触到现在的感觉是 阿里的东西bug太多了，比如我之前给fastjson提交了个非常猎奇的[bug](https://github.com/alibaba/fastjson/issues/497)（到现在都没有回复，对阿里的好感度已经降到底了）

Netflix的这些东西也是搞微服务用的，大约有这些东西Zuul、Eureka、Ribbon、Hystrix、Feign，花了一个下午整了个简单的demo。
<!--more-->

## Eureka
服务发现，一般是有个Eureka Server负责Client的注册，其他没啥功能，需要配合其他工具用

### Server

 pom.xml
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-eureka-server</artifactId>
</dependency>
```

 TestEurekaServerApplication.java

只需加上`@EnableEurekaServer`注解
```java
@SpringBootApplication
@EnableEurekaServer
public class TestEurekaServerApplication {

    public static void main(String[] args) {
        SpringApplication.run(TestEurekaServerApplication.class, args);
    }
}
```
 application.yml

```sql
## 端口
server:
  port: 10001


## 该应用名字
spring:
  application:
    name: test-server

## eureka server相关配置
eureka:
  instance:
    hostname: localhost
  client:
    registerWithEureka: false
    fetchRegistry: false
    serviceUrl:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/

```

启动后打开浏览器是这样的
![01](http://r.loli.io/6NbUne.png)

### Service
我们创建一个服务给远端来调用，这个服务是需要注册到eureka的。主要配置

 pom.xml

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-eureka</artifactId>
</dependency>
```

 TestEurekaServiceApplication.java
创建了个url为/hello的controller方法，我们后面要做的是把这个方法暴露给远程客户端去调用

加上@EnableEurekaClient注解就能注册到Eureka了
```java
@SpringBootApplication
@EnableEurekaClient
public class TestEurekaServiceApplication {

    public static void main(String[] args) {
        SpringApplication.run(TestEurekaServiceApplication.class, args);
    }
}


@RestController
class RemoteHelloController {

    //Eureka会自动注入注册的所有Client信息，不过并没有啥用处
    @Autowired
    private DiscoveryClient client;

    @RequestMapping("hello")
    public String hello() {
        long start = System.currentTimeMillis();
        ServiceInstance instance = client.getLocalServiceInstance();
	// 随机睡眠1000毫秒以内
        try {
            Thread.sleep(new Random().nextInt(1000));
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        long cost = System.currentTimeMillis() - start;
        return "Remote Hello~ " + instance.getHost() + ", " + instance.getServiceId()
              + ", spent " + cost;
    }
}

```
 application.yml

```sql
eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:10001/eureka/

spring:
  application:
    name: test-service

server:
  port: 10002
```

## Client
得有个客户端去调用注册的服务，这个客户端也是要注册到eureka的，和Service在配置上并没有啥不同


```java
@SpringBootApplication
@EnableEurekaClient
public class TestEurekaClientApplication {

    public static void main(String[] args) {
SpringApplication.run(TestEurekaClientApplication.class, args);
    }

    @Autowired
    HelloService helloService;

    @RequestMapping("hello")
    public String hello() {
        return helloService.hello();
    }

    @Autowired
    RestTemplate restTemplate;

    @RequestMapping("ribbonHello")
    public String ribbonHello() {
        return restTemplate.getForEntity("http://test-service/hello", String.class).getBody();
    }
}

```


调用Service的方法有很多种，下面一一列出来

### Ribbon


```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-ribbon</artifactId>
</dependency>
```
Ribbon实际上和Eureka并没有啥关系啦，Ribbon是个类似于负载均衡的东西，只需要引入Ribbon的依赖，然后Ribbon就会自己创建个RestTemplate的实例了，这个RestTemplate还比较高端，和普通的不一样

比如配置文件里这样写
```sql
test-service:
  ribbon:
    listOfServers: localhost:10002
```
然后代码里这样写
```java
restTemplate.getForEntity("http://test-service/hello", String.class).getBody();
```
调用的就是http://localhost:10002/hello了

以下是稍微完整的栗子

```

@SpringBootApplication
@EnableEurekaClient
@RestController
public class TestEurekaClientApplication {

    public static void main(String[] args) {
        SpringApplication.run(TestEurekaClientApplication.class, args);
    }

    @Autowired
    RestTemplate restTemplate;

    @RequestMapping("ribbonHello")
    public String ribbonHello() {
        return restTemplate.getForEntity("http://test-service/hello", String.class).getBody();
    }
}
```

如图：
![02](http://r5.loli.io/v2eeqa.png)

### Zuul
只要引入Zuul依赖，就会自动创建直接可以调用远程服务的mapping path

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-zuul</artifactId>
</dependency>
```

启动日志里会出现如下内容
```sql
2016-02-21 15:41:02.741  INFO 13460 --- [pool-5-thread-1] o.s.c.n.zuul.web.ZuulHandlerMapping      : Mapped URL path [/test-service/**] onto handler of type [class org.springframework.cloud.netflix.zuul.web.ZuulController]
```
然后打开这个url
![03](http://r.loli.io/amIRVr.png)

可以自定义这个mapping path

```sql
zuul:
  ignoredServices: *
  routes:
    test-service: /test/**

```

Zuul还可以当动态路由和反向代理来用，就不多讲了

### Feign

这个好像才有点远程调用的意思了，可以直接把接口映射到某个controller方法上面去，然后调用这个接口就相当于调用远程controller的方法了

```java
@FeignClient("test-service")
interface RemoteHelloService {
    @RequestMapping(value = "hello", method = RequestMethod.GET)
    public String remoteHello();
}
```

可以直接注入这个RemoteHelloService了
```java
@Autowired
RemoteHelloService remoteHelloService;

@RequestMapping("remoteHello")
public String remoteHello() {
    return remoteHelloService.remoteHello();
}

```

如图
![11](http://r5.loli.io/Zfm2Iv.png)


### Hystrix
服务调用出错或者超时的时候可以用这个来解决，还记得上面Service里的hello方法我加了不到一秒的休眠么？

这里直接注入上面Feign的的RemoteHelloService，设置了500毫秒的超时时间，当超时的时候就fallback，去执行timeout方法

```
@RestController
class RemoteHelloController {

    @Autowired
    RemoteHelloService remoteHelloService;

    @RequestMapping("remoteHello")
    // fallbackMethod 似乎不支持被@Configuration的Controller方法
    @HystrixCommand(fallbackMethod = "timeout", commandProperties = {
            @HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds", value = "500")
    })
    public String remoteHello() {
        return remoteHelloService.remoteHello();
    }

    @HystrixCommand
    public String timeout() {
        return "Remote Timeout~";
    }
}
```
如图是超时的情况

![12](http://r6.loli.io/7vayIv.png)


注：更多Hystrix属性见[这里](https://github.com/Netflix/Hystrix/blob/a578774f735eba3f02875ba98eb85db36a47c102/hystrix-contrib/hystrix-javanica/README.md)


Hystrix还有个dashboard，看上去很不错，这里就不介绍啦


有关Spring Cloud更多内容请看[官方文档](http://cloud.spring.io/spring-cloud-static/docs/1.0.x/spring-cloud.html)啦（很多东西文档里都没有。。）