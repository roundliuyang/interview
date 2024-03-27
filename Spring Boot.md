# Spring Boot





## 什么是 Spring Boot



传统的 SSM/SSH 框架组合配置繁琐臃肿，不同项目有很多重复、模板化的配置，严重降低了 Java 工程师的开发效率，而 Spring Boot 可以轻松创建基于 Spring 的、可以独立运行的、生产级的应用程序。通过对Spring 家族和一些第三方库提供一系列自动化配置的 Starter，来使得开发快速搭建一个基于 Spring 的应用程序。

Spring Boot 提供了各种 Starter 启动器，提供标准化的默认配置。例如：

- [`spring-boot-starter-web`](https://mvnrepository.com/artifact/org.springframework.boot/spring-boot-starter-web/2.1.1.RELEASE) 启动器，可以快速配置 Spring MVC 。
- [`mybatis-spring-boot-starter`](https://mvnrepository.com/artifact/org.mybatis.spring.boot/mybatis-spring-boot-starter/1.3.2) 启动器，可以快速配置 MyBatis 。

并且，Spring Boot 基本已经一统 Java 项目的开发，大量的开源项目都实现了其的 Starter 启动器。例如：

- [`incubator-dubbo-spring-boot-project`](https://github.com/apache/incubator-dubbo-spring-boot-project) 启动器，可以快速配置 Dubbo 。
- [`rocketmq-spring-boot-starter`](https://github.com/maihaoche/rocketmq-spring-boot-starter) 启动器，可以快速配置 RocketMQ 。





## SpringBoot 启动流程

SpringApplication#run方法流程如下

- 创建 stopWatch 对象，并启动，用于简单统计 run 启动过程的时长

- **获得 SpringApplicationRunListener 的数组，并启动监听**

  >使用自动配置，获得数组。容器启动、刷新、环境、运行都需要监听器处理相关事件

- **加载属性配置**。执行完成后，所有的 environment 的属性都会加载进来，包括 application.properties 和外部的属性配置

-  **创建 Spring 容器**

- 调用所有初始化类的 **initialize** 方法,**准备 ApplicationContext 对象，主要是初始化它的一些属性。**

- **启动（刷新） Spring 容器**

- 通知 SpringApplicationRunListener 的数组，Spring **容器启动完成**。

- 调用 ApplicationRunner 或者 CommandLineRunner 的运行方法。

  > ApplicationRunner 、CommandLineRunner` 接口的 `Component` 会在所有 `Spring Beans `都初始化之后执行，非常适合在应用程序启动之初进行一些数据初始化的工作。

- 通知 **SpringApplicationRunListener** 的数组，Spring 容器运行中。







## Spring Boot 的核心注解



- @SpringBootConfiguration：组合了 @Configuration 注解，实现配置文件的功能。
- @EnableAutoConfiguration：打开自动配置的功能，也可以关闭某个自动配置的选项，如关闭数据源自动配置功能-@SpringBootApplication(exclude = { DataSourceAutoConfiguration.class })。
- @ComponentScan：Spring组件扫描。



## Spring Boot 中如何实现定时任务



定时任务也是一个常见的需求，Spring Boot 中对于定时任务的支持主要还是来自 Spring 框架。

在 Spring Boot 中使用定时任务主要有两种不同的方式，一个就是使用 Spring 中的 @Scheduled 注解，另一个则是使用第三方框架 Quartz。

使用 Spring 中的 @Scheduled 的方式主要通过 @Scheduled 注解来实现。

使用 Quartz ，则按照 Quartz 的方式，定义 Job 和 Trigger 即可。





## 如何统一引入 Spring Boot 版本



**目前有两种方式**

① 方式一：继承 `spring-boot-starter-parent` 项目。配置代码如下：

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>1.5.1.RELEASE</version>
</parent>
```

② 方式二：导入 spring-boot-dependencies 项目依赖。配置代码如下：

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-dependencies</artifactId>
            <version>1.5.1.RELEASE</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

**如何选择**

因为一般我们的项目中，都有项目自己的 Maven parent 项目，所以【方式一】显然会存在冲突。所以实际场景下，推荐使用【方式二】。

另外，在使用 Spring Cloud 的时候，也可以使用这样的方式。





## Spring Boot 如何定义多套不同环境配置



可以参考 [《Spring Boot 教程 - Spring Boot Profiles 实现多环境下配置切换》](https://blog.csdn.net/top_code/article/details/78570047) 一文。

但是，需要考虑一个问题，生产环境的配置文件的安全性，显然我们不能且不应该把生产的配置放到项目的 Git 仓库中进行管理。那么应该怎么办呢？

- 方案一，生产环境的配置文件放在生产环境的服务器中，以 `java -jar myproject.jar --spring.config.location=/xxx/yyy/application-prod.properties` 命令，设置 参数 `spring.config.location` 指向配置文件。
- 方案二，使用 Jenkins 在执行打包，配置上 Maven Profile 功能，使用服务器上的配置文件。😈 整体来说，和【方案一】的差异是，将配置文件打包进了 Jar 包中。
- 方案三，使用配置中心。





## Spring Boot 配置加载顺序



1. `spring-boot-devtools` 依赖的 `spring-boot-devtools.properties` 配置文件。
2. 单元测试上的 `@TestPropertySource` 和 `@SpringBootTest` 注解指定的参数。
3. 命令行指定的参数。例如 `java -jar springboot.jar --server.port=9090` 。
4. 命令行中的 `spring.application.json` 指定参数。例如 `java -Dspring.application.json='{"name":"Java"}' -jar springboot.jar` 。
5. ServletConfig 初始化参数。
6. ServletContext 初始化参数。
7. JNDI 参数。例如 `java:comp/env` 。
8. Java 系统变量，即 `System#getProperties()` 方法对应的。
9. 操作系统环境变量。
10. RandomValuePropertySource 配置的 `random.*` 属性对应的值。
11. Jar **外部**的带指定 profile 的 application 配置文件。例如 `application-{profile}.yaml` 。
12. Jar **内部**的带指定 profile 的 application 配置文件。例如 `application-{profile}.yaml` 。
13. Jar **外部** application 配置文件。例如 `application.yaml` 。
14. Jar **内部** application 配置文件。例如 `application.yaml` 。
15. 在自定义的 `@Configuration` 类中定于的 `@PropertySource` 。
16. 启动的 main 方法中，定义的默认配置。即通过 `SpringApplication#setDefaultProperties(Map<String, Object> defaultProperties)` 方法进行设置。

每一种配置方式的详细说明，可以看看 《Spring Boot 参考指南（外部化配置）》 。







## 什么是 Spring Boot 自动配置

```java
// 开启（刷新）Spring 容器
refresh(context);
```

**启动容器的时候进行自动配置。**

`@EnableAutoConfiguration` 注解，打开 Spring Boot 自动配置的功能。

如下是一个比较简单的总结：

1. Spring Boot 在启动时扫描项目所依赖的 jar 包，寻找包含`spring.factories` 文件的 jar 包。
2. 根据 `spring.factories` 配置加载 AutoConfigure 类。
3. 根据 [`@Conditional` 等条件注解](http://svip.iocoder.cn/Spring-Boot/Interview/Spring Boot 条件注解) 的条件，进行自动配置并将 Bean 注入 Spring IoC 中。



**简单流程**

-  开启（刷新）Spring 容器   refresh
- ......
- AutoConfigurationImportSelector#process方法
- 通过 SpringFactoriesLoader 类提供的方法加载类路径中META-INF目录下的spring.factories 文件中针对EnableAutoConfiguration的 注册配置类 
- 





## Spring Boot 有几种读取配置的方式



Spring Boot 目前支持 **2** 种读取配置：

1. `@Value` 注解，读取配置到属性。

   > 另外，支持和 `@PropertySource` 注解一起使用，指定使用的配置文件。

2. `@ConfigurationProperties` 注解，读取配置到类上

   >另外，支持和 `@PropertySource` 注解一起使用，指定使用的配置文件。





## Spring Boot 启动的时候运行一些特殊的代码



如果需要在 SpringApplication 启动后执行一些特殊的代码，你可以实现 ApplicationRunner 或 CommandLineRunner 接口，这两个接口工作方式相同，都只提供单一的 run 方法，该方法仅在 `SpringApplication#run(...)` 方法**完成之前调用**。

一般情况下，我们不太会使用该功能。如果真需要，胖友可以详细看看 [《使用 ApplicationRunner 或 CommandLineRunner 》](https://qbgbook.gitbooks.io/spring-boot-reference-guide-zh/IV. Spring Boot features/23.8 Using the ApplicationRunner or CommandLineRunner.html) 。







## Spring Boot 集成



**如何集成 Spring Boot 和 MyBatis**

>引入 mybatis-spring-boot-starter 的依赖。



**如何集成 Spring Boot 和 RabbitMQ** 

>引入 spring-boot-starter-amqp 的依赖



**如何集成 Spring Boot 和 Kafka** 

>引入 spring-kafka 的依赖



**如何集成 Spring Boot 和 RocketMQ**

>引入 rocketmq-spring-boot 的依赖

