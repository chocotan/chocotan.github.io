---
title: Spring Cloud学习笔记——Spring Cloud Config的使用(1)
date: 2016-01-29 00:56:13
tags: [spring,java]
categories: java
---


最近迷上了Spring Cloud，动态路由、负载均衡、服务发现，各种功能，觉得可以替换我司目前使用的任务调度和微服务了，我司任务调度用的是tb-schedule(基于zookeeper)，微服务是自己用dubbo实现的。

实际开发中对于任务分片用的很少，而是对于查询出数据后，如何高效地处理这些数据的问题。tb-schedule想要多线程的话，只能配置多个任务项，导致查询数据的时候必须根据任务项去区分开来。
<!--more-->

```sql
-- 线程0，查询出来处理
select * from tab01 where flag=0
-- 线程1，查询出来处理
select * from tab01 where flag=1
-- 线程2，查询出来处理
select * from tab01 where flag=2
```
而我想要的并不是查询时分多个线程，而是一起查询，多线程去处理。


于是自己想开发一个任务调度框架了，不过先要熟悉一下spring cloud到底有哪些内容，于是照着英文文档围观了一下



##spring-cloud-config-server
如果配置是写死在xml或者properties文件里，且需要修改这个配置的时候，咋办？停掉修改打包上传么？用了这个config-server的话，可以不需要改代码方便的修改配置了


pom.xml中增加如下依赖

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-server</artifactId>
</dependency>
```

```java
@SpringBootApplication
@EnableConfigServer
public class ConfigServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(ConfigServerApplication.class, args);
    }
}

```

`application.yml`

```yml
server:
  port: 8888

management:
  context-path: /admin

spring:
  cloud:
    config:
      server:
        git:
          uri: /home/choco/soft/gittest/
```

spring-cloud-config是用git作为配置仓库的，git.url可以是本地的git目录，也可以是远程的git目录，会自动加载该目录下的properties文件，


我的git仓库里有
`test.properties`
```
choco.name=chocotan
choco.password=123456
```

启动后打开浏览器`http://localhost:8888/test/master`
大约是这样的格式 [ip:address]/[filename]/[branch]
返回

```json
{
    "name": "test",
    "profiles": ["master"],
    "label": null,
    "version": "2ab9d0ab4f9d88e0eed31702e038d9ec11d7ae16",
    "propertySources": [{
        "name": "/home/choco/soft/gittest/test.properties",
        "source": {
            "choco.password": "123456",
            "choco.name": "chocotan"
        }
    }]
}
```


嗯 大约完成config-server的配置了
