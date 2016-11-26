---
title: spring boot中@EnableXXX注解的做法
date: 2016-11-23 21:51:00
tags:
---

接触springboot也有一阵子了，之前就觉得@EnableXX的注解很酷炫，于是就简单研究了一下，可以给自己的模块加上这个功能

以下是一个简单的实现，只要在spring boot启动类上加上@EnableHello就能在启动完的时候输出"Hello Spring Boot"了


首先要有一个@EnableHello注解类

```java
package io.loli.demo.enable;

import org.springframework.context.annotation.Import;

import java.lang.annotation.*;

/**
 * @author choco
 */

@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@Import({HelloConfigurationSelector.class})
public @interface EnableHello {
}

```

这里用到了HelloConfigurationSelector

```java
package io.loli.demo.enable;

import org.springframework.context.annotation.ImportSelector;
import org.springframework.core.type.AnnotationMetadata;

/**
 * @author choco
 */
public class HelloConfigurationSelector implements ImportSelector {
    @Override
    public String[] selectImports(AnnotationMetadata annotationMetadata) {
        return new String[]{HelloConfiguration.class.getName()};
    }
}

```

这个ImportSelector接口挺有意思的，大家有兴趣可以深入研究下


```java
package io.loli.demo.enable;

import org.springframework.boot.CommandLineRunner;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

/**
 * @author choco
 */
@Configuration
@ConfigurationProperties(prefix = "hello")
public class HelloConfiguration {
    private String message;

    @Bean
    public CommandLineRunner commandLineRunner() {
        return args -> System.out.println(message);
    }

    public void setMessage(String message) {
        this.message = message;
    }
}

```


如上就是这个小工程的三个主角了，打包install之后，就可以在别的工程里用了

再新建一个Springboot工程，在启动类上添加@EnableHello注解

```java
package io.loli.demo.test;

import io.loli.demo.enable.EnableHello;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
@EnableHello
public class HelloTestApplication {

    public static void main(String[] args) {
        SpringApplication.run(HelloTestApplication.class, args);
    }

}

```

```
hello.message=Hello Spring Boot
```

运行这个类，就会输出hello.message中的Hello Spring Boot字符串了




