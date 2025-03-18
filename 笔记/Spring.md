# SpringBoot

## 特性

快速的构建能力，起步即可依赖，内嵌容器支持，Actuator监控

1. <font color='Chestnut Red'>**快速的构建能力**</font>

`SpringBoot`提供了一堆`Starters`用于快速构建项目，它包含了一系列可集成到应用中的依赖包，可直接在`pom`中引用而不用到处去找，例如创建一个`web`项目，只要引入`spring-boot-starter-web`依赖即可。

2. <font color='Chestnut Red'>**起步即可依赖**</font>

`SpringBoot` 在新建项目时即可勾选依赖项，在项目初始化时就把相关依赖加进去，你需要数据库就把数据库相关 `starter` 加进去，需要单元测试支持，就把单元测试相关 `starter` 加进去，这样就大大缩短了去查询依赖的时间。

3. <font color='Chestnut Red'>**内嵌容器支持**</font>

`SpringBoot `内嵌了` Tomcat、Jetty、Undertow` 三种容器，也就是说，以往用 `Spring` 构建 `web` 项目还要配置 `Tomcat` 等容器，现在不用了。其默认嵌入的容器是**`Tomcat`** 默认端口是` 8080`

4. <font color='Chestnut Red'>**Actuator 监控**</font>

**`SpringBoot` 自带了 `Actuator` 监控功能，主要用于提供对应用程序监控，以及控制的能力**，比如监控应用程序的运行状况，或者内存、线程池、`Http` 请求统计等，同时还提供了关闭应用程序等功能。

## SpringBoot 启动流程

1. **加载启动类**

      - 启动类是标准 Java 类，包含 `main` 方法，标记 `@SpringBootApplication` 注解。

      - `@SpringBootApplication` 组合了三个核心注解：
        - `@SpringBootConfiguration`：标识启动类是配置类。
        - `@EnableAutoConfiguration`：开启自动配置。
        - `@ComponentScan`：扫描当前包及子包下的组件。


2. **初始化 `SpringApplication` 实例**
      - 调用 `SpringApplication.run(启动类.class, args)` 时：
        1. **推断应用类型**：根据类路径是否存在 `DispatcherServlet` 等类，判断是 Web 应用（Servlet/Reactive）还是非 Web 应用。
        2. **加载配置源**：通过 `SpringFactoriesLoader` 加载 `META-INF/spring.factories` 中的配置类，初始化监听器（`ApplicationListener`）和初始化器（`ApplicationContextInitializer`）。
        3. **准备环境（Environment）**：加载外部属性（如 `application.yml`），合并命令行参数、系统环境变量等**按优先级覆盖**配置。


3. **创建并准备应用上下文（ApplicationContext）**

      - **实例化上下文类型**（例如）：
        - Web：`AnnotationConfigServletWebServerApplicationContext`
        - 非 Web：`AnnotationConfigApplicationContext`

      - **触发初始化器**：执行注册的 `ApplicationContextInitializer` 自定义上下文配置。


4. **刷新上下文（`refresh()` 核心阶段）**

   通过 `AbstractApplicationContext.refresh()` 完成关键步骤：

   > 1. **准备 BeanFactory**：
   >    创建 `DefaultListableBeanFactory`，注册核心后处理器（如 `ConfigurationClassPostProcessor`）。
   > 2. 加载 Bean 定义：
   >    - 扫描 `@Component`、`@Service`、`@Repository` 等注解的类。
   >    - 自动配置阶段：
   >      - 从 `spring.factories` 加载 `EnableAutoConfiguration` 类。
   >      - 过滤排除项（`@EnableAutoConfiguration.exclude`）。
   >      - 通过条件注解（如 `@ConditionalOnClass`）决定是否应用配置。
   > 3. **调用 BeanFactoryPostProcessor**：例如处理 `@Configuration` 类中的 `@Bean` 方法，或加载属性占位符（`@Value`）。
   > 4. **注册 BeanPostProcessor**：准备好处理 Bean 初始化前后的逻辑（如 `@Autowired` 注入、`@PostConstruct` 回调）。
   > 5. 初始化单例 Bean：
   >    - 实例化所有非懒加载的单例 Bean，完成依赖注入和初始化（`@PostConstruct`）。
   >    - 启动内嵌服务器（仅限 Web 应用）：
   >      - 检测依赖中的服务器类型（如 Tomcat、Jetty）。
   >      - 创建 `WebServer` 并绑定端口，但未开始监听（需要在后续阶段启动）。
   > 6. **发布 `ContextRefreshedEvent` 事件**：通知监听者上下文刷新完成。

5. **执行启动扩展点**

      - **`ApplicationRunner` 和 `CommandLineRunner`**：
        - 所有实现类按 `@Order` 顺序执行 `run()` 方法，支持启动后预加载逻辑。

      - **生命周期回调**：
        - `@PostConstruct`：在 Bean 初始化期间调用。
        - `SmartLifecycle` 接口：控制更精确的生命阶段（如启动、停止）。


6. **启动 Web 服务器并监听请求（Web 场景）**

      - **启用服务器监听**：
        - 在上下文刷新的最后阶段，调用 `WebServer.start()` 开始监听端口。
        - **触发 `ServletWebServerInitializedEvent`**：通知服务器已准备就绪。

      - **初始化 MVC 组件**：
        - 加载控制器（`@RestController`），注册过滤器（`Filter`）、拦截器等。


7. **启动完成**

      - **发布 `ApplicationStartedEvent` 和 `ApplicationReadyEvent`**：
        - 区分“启动中”与“完全就绪”状态。

      - **控制台输出 Banner**（默认开启）和启动日志。


8. **等待请求 & 运行阶段**

      - 应用进入持续运行状态，处理传入的 HTTP 请求或执行定时任务等。

      - 可通过 `Ctrl+C` 或 `SIGTERM` 优雅关闭，触发 `ApplicationContext.close()`, 销毁 Bean 并释放资源。


---

### **关键补充说明**
1. **事件驱动机制**：
   Spring Boot 通过 `ApplicationEvent` 和 `ApplicationListener` 实现启动过程的可观测性（如 `ApplicationStartingEvent`, `ApplicationPreparedEvent`）。
2. **条件装配（Conditional）**：
   自动配置根据类路径、Bean 存在性、属性值等条件动态生效（例如 `@ConditionalOnMissingBean` 允许用户覆盖默认配置）。
3. **外部化配置的优先级**：
   命令行参数 > 系统环境变量 > `application-{profile}.yml` > `application.yml` > 默认属性。
4. **内嵌服务器的选择逻辑**：
   根据引入的依赖（如 `spring-boot-starter-web` 默认选用 Tomcat）自动适配服务器类型。

---

### **流程图简化描述**
```
加载启动类 → 初始化 SpringApplication → 准备 Environment → 创建上下文 → 刷新上下文（加载配置、创建 Bean、启动服务器） → 执行 Runner → 完全运行
```

此版本补充了 `SpringApplication` 初始化、条件装配细节、事件机制及上下文刷新的完整阶段，修正了原始回答中较为模糊的环节。

## 约定大于配置

> `SpringBoot`约定大于配置的理解

可以从两个方面来理解：

1. 开发人员仅需规定应用中不符合约定的部分

2. 在没有规定配置的地方，采用默认配置，以力求最简配置为核心思想

总的来说，上面两条都遵循了推荐默认配置的思想。当存在特殊需求的时候，也可以自定义配置。这样可以大大减少配置工作，这就是所谓的“约定”。

> `SpringBoot`中有哪些约定？

1. `Maven`的目录结构。默认有`resources`文件夹,存放资源配置文件。`src-main-resources,src-main-java`。默认的编译生成的类都在`targe`文件夹下面

2. `springboot`默认的配置文件必须只能是`application.`命名的`yml`文件或者`properties`文件，且唯一

3. `application.yml`中默认属性。数据库连接信息必须是以`spring: datasource: `为前缀；多环境配置。该属性可以根据运行环境自动读取不同的配置文件；端口号、请求路径等

##核心配置文件

`SpringBoot`的核心配置文件有**`application`和`bootstarp`配置文件。**
**区别**
`application`文件主要用于`Springboot`自动化配置文件。
`bootstarp`文件主要有以下几种用途：

- 使用`Spring Cloud Config`注册时，需要在`bootStarp`配置文件中添加链接到配置中心的配置属性来加载外部配置中心的配置信息。
- 一些固定的不能被覆盖的属性
- 一些加密/解密的场景

**都有什么格式**

- `.properties` 和 `.yml`
- `.yml`采取的是缩进的格式 不支持`@PeopertySource`注解导入配置

## 核心注解

`SpringBoot`的核心注解是`@SpringBootApplication`，可以看作是 `@Configuration`、`@EnableAutoConfiguration`、`@ComponentScan` 注解的集合。

- `@EnableAutoConfiguration`：打开自动配置的功能，也可以关闭某个自动配置的选项，如关闭数据源自动配置功能
- `@ComponentScan`： 扫描被`@Component` (`@Service`,`@Controller`)注解的 bean，注解默认会扫描该类所在的包下所有的类。
- `@Configuration`：允许在 Spring 上下文中注册额外的 bean 或导入其他配置类

##配置文件的加载顺序

**内部配置文件**

（`按照优先级从高到低的顺序`）：

1. 先去项目根目录找`config`文件夹下找配置文件
2. 再去根目录下找配置文件
3. 去`resources`下找`cofnig`文件夹下找配置文件
4. 去`resources`下找配置文件

如果高优先级中配置文件属性与低优先级配置文件不冲突的属性，则会共同存在—`互补配置`。

这里说的配置文件，都还是项目里面。最终都会被打进`jar`包里面的，需要注意。

> 1、如果同一个目录下，有`application.yml`也有`application.properties`，默认先读取`application.properties`。
> 2、如果同一个配置属性，在多个配置文件都配置了，默认使用第`1`个读取到的，后面读取的不覆盖前面读取到的。
> 3、创建`SpringBoot`项目时，一般的配置文件放置在“项目的`resources`目录下”

项目打包好以后，我们可以使用命令行参数的形式，启动项目的时候来指定配置文件的新位置。
指定配置文件和默认加载的这些配置文件共同起作用形成互补配置。

**外部配置**

1. 命令行参数（所以我们`java -jar`启动时指定的参数优先级最高）
   所有的配置都可以在命令行上进行指定；
   多个配置用空格分开； `--配置项=值`

2. Java系统属性（`System.getProperties()`）
   由此可见，`Spring`启动的时候，默认会把系统的很多属性都默认加载进来

   自定义`key`的时候，应该去避免和系统自带的`key`重名，否则不起作用。

3. 操作系统环境变量（比如操作系统的`username`等等）

4. `RandomValuePropertySource`配置的`random.*`属性值

5. `jar包外部`的`application-{profile}.properties`配置文件

6. `jar包内部`的`application-{profile}.properties`配置文件

7. `jar包外部`的`application.properties`配置文件（此级别在测试环境经常使用。比如就在jar包同级目录放置一个配置文件，就内覆盖jar包内部所有的配置文件了）

8. `jar包内部`的`application.properties`配置文件

   由`jar`包外向`jar`包内进行寻找，优先加载待`profile`的，再加载不带`profile`的

9. `@Configuration`注解类上的`@PropertySource`（手动指定导入外部配置文件）

10. 通过`SpringApplication.setDefaultProperties`指定的默认属性，自己程序代码里设置，优先级最低

<font color='Chestnut Red'>**不管内部、外部配置，形成的都是互补配置，都会加载**</font>

## 配置信息读取

默认读取配置文件 `application.properties` 或者 `application.yml` 中的配置信息，两种不同的文件类型，对应的内部配置方式也不太一样

**配置读取**

1. `Environment` 读取
   所有的配置信息，都会加载到`Environment`实体中，因此可以通过这个对象来获取系统的配置，通过这种方式不仅可以获取`application.yml`配置信息，还可以获取更多的系统信息

2. `@Value` 注解方式
   `@Value`注解可以将配置信息注入到`Bean`的属性，但需要注意
   - **如果配置信息不存在会报错**，可以在引用变量时给它赋一个默认值，以确保即使在未正确配值的情况下，程序依然能够正常运行。例如：@Value("${env.var:我是小富}")
   - **@Value 注解加到静态变量上**，这样做是无法获取属性值的。静态变量是类的属性，并不属于对象的属性，而 Spring是基于对象的属性进行依赖注入的，类在应用启动时静态变量就被初始化，此时 Bean还未被实例化，因此不可能通过 @Value 注入属性值。
   - **@Value 注解加到 final 关键字**上同样也无法获取属性值，因为 final 变量必须在构造方法中进行初始化，并且一旦被赋值便不能再次更改。而 @Value 注解是在 bean 实例化之后才进行属性注入的，因此无法在构造方法中初始化 final 变量
   - 只有由 Spring 管理的 bean 中使用 @Value注解才会生效。而对于普通的POJO类，无法使用 @Value注解进行属性注入。

3. 对象映射方式

   - 通过注解`ConfigurationProperties`来制定配置的前缀
   - 通过`Bean`的属性名，补上前缀，来完整定位配置信息的`Key`，并获取`Value`赋值给这个`Bean`

##常用的stater。如何理解stater

`spring-boot-starter-web `

引入了这个`maven`依赖，基本就满足了日常的`web`接口开发，而不使用`springboot`时，需要引入`spring-web、spring-webmvc、spring-aop`等等来支持项目开发。

`starter`可以当成是一个`maven`依赖组，引入这个组名就引入了相关的依赖和一些初始化的配置。

![img](https://gitee.com/qc_faith/picture/raw/master/image/202211051829363.png)

## 启动时运行特定的代码

实现接口 **`ApplicationRunner`** 或者 **`CommandLineRunner`**，并重写其`run `方法。

**`CommandLineRunner`**：启动获取命令行参数。

```java
@FunctionalInterface
public interface CommandLineRunner {
	/**
	 * Callback used to run the bean.
	 * @param args incoming main method arguments
	 * @throws Exception on error
	 */
	void run(String... args) throws Exception;
}
```

**`ApplicationRunner`**：启动获取应用启动的时候参数。

```java
@FunctionalInterface
public interface ApplicationRunner {
	/**
	 * Callback used to run the bean.
	 * @param args incoming application arguments
	 * @throws Exception on error
	 */
	void run(ApplicationArguments args) throws Exception; 
}
```

如果启动的时候有多个`ApplicationRunner`和`CommandLineRunner`，想控制它们的启动顺序，可以实现 `org.springframework.core.Ordered`接口或者使用 `@Order`注解。

**区别**

- `CommandLineRunner`的方法参数是原始的参数，未做任何处理；
- `ApplicationRunner`的参数为`ApplicationArguments`对象，是对原始参数的进一步封装。`ApplicationArguments`是对参数（主要针对`main`方法）做了进一步的处理，可以解析`--name=value`的参数，我们就可根据name获取value。

## @SpringBootApplication注解

是 Spring Boot 中的一个核心注解，它用于标识一个主要的 Spring Boot 应用程序类。

1. **组合注解：**SpringBootApplication 实际上是一个组合注解，它包含了多个其他的注解，用于快速配置和启动一个 Spring Boot 应用程序。具体来说，它包括了以下三个注解的功能： 
   - @SpringBootConfiguration：标识该类为 Spring Boot 配置类，类似于传统 Spring 的 @Configuration 注解。
   - @EnableAutoConfiguration：启用自动配置，让 Spring Boot 根据类路径中的依赖来自动配置应用程序的各个组件。
   - @ComponentScan：扫描当前包及其子包，查找带有 Spring 相关注解（如 @Component、@Service、@Controller 等）的类，将它们注册为 Spring 的 Bean。
2. <font style="color:rgb(78, 84, 91);"> </font>**主程序入口：**在一个 Spring Boot 应用程序中，@SpringBootApplication 注解通常被标注在应用程序的主类上，即包含 main 方法的类。它是应用程序的入口点，通过执行 main 方法来启动 Spring Boot 应用。
3. <font style="color:rgb(78, 84, 91);"> </font>**约定大于配置：**@SpringBootApplication 注解代表了 Spring Boot 的约定大于配置的思想。它默认会启用自动配置，扫描并注册需要的组件，从而使得应用程序的配置过程变得简单，并且能够快速搭建一个功能完备的 Spring Boot 应用。
4. <font style="color:rgb(78, 84, 91);"> </font>**配置扩展：**<font style="color:rgb(78, 84, 91);">  </font>尽管 @SpringBootApplication 注解已经包含了许多默认的配置，你仍然可以在应用程序中添加自己的配置类、自定义 Bean 和其他相关组件，来进一步定制和扩展应用程序的行为。

## 自动配置原理

Spring Boot的自动装配是通过`@EnableAutoConfiguration`注解实现的。Spring Boot通过条件化配置（Conditional Configuration）和SPI（Service Provider Interface）机制，实现了一种基于约定优于配置的自动化配置策略。

1. **@EnableAutoConfiguration注解：** Spring Boot应用中通常会在启动类上使用`@SpringBootApplication`注解，而`@SpringBootApplication`注解本身包含了`@EnableAutoConfiguration`注解。`@EnableAutoConfiguration`注解是Spring Boot的关键注解之一，它启用了Spring Boot的自动配置机制。
2. **@Conditional注解：** Spring Boot中的很多自动配置类都使用了条件注解（`@Conditional`）进行条件判断。条件注解允许根据满足一定条件的情况来决定是否应用某个配置。
3. **spring.factories文件：** Spring Boot的自动配置信息通常是通过`META-INF/spring.factories`文件中的配置来定义的。这个文件位于项目的classpath下，其中包含了各种自动配置类的配置信息。每个自动配置类都会在`spring.factories`中定义一个`EnableAutoConfiguration`的键，指向该自动配置类的类路径。
4. **SPI机制：** Spring Boot利用了Java的SPI（Service Provider Interface）机制。在`spring.factories`文件中，配置了各种自动配置类对应的`EnableAutoConfiguration`实现类。Spring Boot在启动时，会通过SPI机制加载这些实现类，并进行相应的自动配置。
5. **自动配置类：** 自动配置类通常包含了一些`@Configuration`注解和`@Conditional`注解，以及各种`@Bean`定义。这些自动配置类的目的是根据条件判断是否需要自动配置一些Bean，以及如何配置这些Bean。

## 可同时处理的请求数

SpringBoot默认的内嵌容器是Tomcat，即程序实际上运行在Tomcat里。与其说SpringBoot可以处理多少请求，到不如说Tomcat可以处理多少请求。

在SpringBoot中处理请求数量相关的参数有四个：


+ **server.tomcat.threads.min-spare**：最少的工作线程数，默认大小是10。该参数相当于长期工，如果并发请求的数量达不到10，就会依次使用这几个线程去处理请求。
+ **server.tomcat.threads.max**：最多的工作线程数，默认大小是200。该参数相当于临时工，如果并发请求的数量在10到200之间，就会使用这些临时工线程进行处理。
+ **server.tomcat.max-connections**：最大连接数，默认大小是8192。表示Tomcat可以处理的最大请求数量，超过8192的请求就会被放入到等待队列。
+ **server.tomcat.accept-count**：等待队列的长度，默认大小是100。

也就是说，SpringBoot 同时所能处理的最大请求数量是 max-connections + accept-count，超过该数量的请求直接就会被丢掉。

# 事务

## @Transactional注意事项

Java中异常的基类为Throwable，有两个子类Exception与Errors。RuntimeException就是Exception的子类，只有RuntimeException才会进行回滚；

- `Spring`默认情况会对`(RuntimeException)`及其子类来进行回滚，在遇见`Exception`或者`Checked`异常(`ClassNotFoundException/NoSuchMetodException`)的时候则不会进行回滚操作。要求我们在自定义异常的时候，让自定义的异常继承自RuntimeException，这样抛出的时候才会被Spring默认的事务处理准确处理。
- `@Transactional`注解只能被应用到`public`方法上，这是由`Spring Aop`本质决定的。

## @Transactional注解属性含义

- `propagation`: 设置可选的事务传播行为
- `isolation`: 可选的事务隔离级别设置
- `readOnly`: 读写或只读事务，默认读写
- `timeout`: 事务超时时间设置
- `rollbackFor`: 导致事务回滚的异常类数组
- `rollbackForClassName`: 导致事务回滚的异常类名称数组
- `noRollbackFor`: 不会导致事务回滚的异常类数组
- `noRollbackForClassName`: 不会导致事务回滚的异常类名称数组

## 事务传播行为

所谓事务的传播行为是指，如果在开始当前事务之前，一个事务上下文已经存在，此时有若干选项可以指定一个事务性方法的执行行为。在`TransactionDefinition`定义中包括了如下几个表示传播行为的常量：

1. `REQUIRED`：如果当前上下文中存在事务，那么加入该事务，如果不存在事务，创建一个事务，这是默认的传播属性值。<font color='Chestnut Red'>**(默认)**</font>
2. `SUPPORTS`：如果当前上下文存在事务，就加入事务，如果不存在事务，则使用非事务的方式执行。
3. `NOT_SUPPORTED`：如果当前上下文中存在事务，则挂起当前事务，然后新的方法在没有事务的环境中执行。
4. `REQUIREDS_NEW`：每次都会新建一个事务，并且同时将上下文中的事务挂起，执行当前新建事务完成以后，上下文事务恢复再执行。
5. `MANDATORY`：一定要有事务。如果当前存在事务，则加入该事务；如果当前没有事务，则抛出异常。
6. `NEVER`：如果当前上下文中存在事务，则抛出异常，否则在无事务环境上执行代码。
7. `NESTED`：如果当前上下文中存在事务，则嵌套事务执行，如果不存在事务，则新建事务。

> NEXSTED 嵌套事务呈现父子事务概念，二者之间是有关联的，核心思想就是子事务不会独立提交，而是取决于父事务，当父事务提交，那么子事务才会随之提交；如果父事务回滚，那么子事务也回滚。与此相反，PROPAGATION_REQUIRES_NEW 的内层事务，会立即提交，与外层毫无关联。
>
> 子事务又有自己的特性，那就是可以独立进行回滚，不会引发父事务整体的回滚(当然需要try catch子事务，避免异常传递至父层事务，如果没有，则也会引发父事务整体回滚)。

## 事务失效

在`Spring`中，事务有两种实现方式

编程式事务管理：编程式事务管理使用`TransactionTemplate`或者直接使用底层的`PlatformTransactionManager`。对于编程式事务管理，`Spring`推荐使用`TransactionTemplate`。

声明式事务管理： 建立在`AOP`之上的。其本质是对方法前后进行拦截，然后在目标方法开始之前创建或者加入一个事务，在执行完目标方法之后根据执行情况提交或者回滚事务。

声明式事务管理不需要入侵代码，通过`@Transactional`就可以进行事务操作，更快捷而且简单。推荐使用

### 失效

<img src="https://gitee.com/qc_faith/picture/raw/master/image/202305301602537.png" alt="image-20230530160230417" style="zoom:33%;" />

1. `Transactional`注解标注方法修饰符为非`public`时，`@Transactional`注解将会不起作用。

   > Spring的事务代理通常是通过Java动态代理或CGLIB动态代理生成的，私有方法无法被代理，因此事务将无效。
   >
   > 在`AbstractFallbackTransactionAttributeSource`类的`computeTransactionAttribute`方法中有个判断，如果目标方法不是public，则`TransactionAttribute`返回null，即不支持事务。

   <img src="https://gitee.com/qc_faith/picture/raw/master/image/202305301643911.png" alt="image-20230530164311839" style="zoom: 33%;" />

2. 方法被定义成了final的，这样会导致事务失效。

   > spring事务底层使用了aop，也就是通过jdk或者cglib帮我们生成了代理类，在代理类中实现的事务功能。
   >
   > 但如果某个方法用final修饰了，那么在它的代理类中，就无法重写该方法，而添加事务功能。

   **如果某个方法是static的，同样无法通过动态代理，变成事务方法**：

   > 因为不管是jdk的动态代理还是cglib的动态代理，都是要通过代理的方式获取到代理的具体对象，而static是属于类。所以静态方法是不能被重写的，正因为不能被重写，所以动态代理也不成立。因为不管是jdk和[cglib动态代理](https://so.csdn.net/so/search?q=cglib动态代理&spm=1001.2101.3001.7020)，都是需要实现或者重写方法的。

3. 方法的内部调用。

   ~~~java
   @Service
   public class UserService {
   
       @Autowired
       private UserMapper userMapper;
   
       @Transactional
       public void add(UserModel userModel) {
           userMapper.insertUser(userModel);
         // 内部直接调用
           updateStatus(userModel);
       }
   
       @Transactional
       public void updateStatus(UserModel userModel) {
           doSameThing();
       }
   }
   ~~~

   解决方式：

   1. 把调用方法移到别的类中
   2. 在类中注入自己
   3. 在该Service类中使用`AopContext.currentProxy()`获取代理对象

4. 未被 Spring 管理

5. 多线程调用，两个方法不在同一个线程中，获取到的数据库连接不一样的。

   > Spring 的事务是通过数据库连接来实现的。当前线程中保存了一个map，key是数据源，value是数据库连接。
   >
   > 我们说的同一个事务，其实是指同一个数据库连接，只有拥有同一个数据库连接才能同时提交和回滚。如果在不同的线程，拿到的数据库连接肯定是不一样的，所以是不同的事务。

6. 表的存储引擎为`MyISAM`是没有事务的，需要使用`InnoDB`

7. 未开启事务

8. 错误的传播特性：`@Transactional` 的 `propagation `属性设置错误，目前只有这三种传播特性才会创建新事务：REQUIRED，REQUIRES_NEW，NESTED。

9. 加事务的方法中手动`try...catch`住了异常，只有将异常抛出来(*无论是主动还是被动*)事务才能回滚

10. `Spring`事务默认回滚的是`RunTimeException`运行时异常，对于普通的Exception（非运行时异常），他不会回滚。可以指定回滚异常

11. 嵌套事务回滚多了

    > 将内部嵌套事务放在try/catch中，并且不继续往上抛异常。这样就能保证，如果内部嵌套事务中出现异常，只回滚内部事务，而不影响外部事务。

12. `@Transactional` 的 `rollbackFor `属性设置错误

~~~java
@Transactional(rollbackFor = BusinessException.class)
public void add(UserModel userModel) throws Exception {
  saveData(userModel);
  updateData(userModel);
}
// 如果在执行上面这段代码时报错了，抛了SqlException、DuplicateKeyException等异常。而BusinessException是我们自定义的异常，报错的异常不属于BusinessException，所以事务也不会回滚。

// 一般情况下建议，将该参数设置成：Exception或Throwable。因为如果使用默认值，一旦程序抛出了Exception，事务不会回滚，这会出现很大的bug。
~~~

# 注解区分

1. <font color='Chestnut Red'>**@Configuration 和 @Component 的区别**</font>

- `@Configuration`: 用于定义配置类，通常与 `@Bean` 注解一起使用，用于声明 bean 的创建和配置信息。配置类中的方法可以包含业务逻辑，同时通过 `@Bean` 注解将方法返回的对象注册为 Spring 容器中的 bean。

  `@Configuration` 类的代理方式和普通 Spring bean 不同。Spring 通过 CGLIB（Code Generation Library）库创建一个子类来代理 `@Configuration` 类，这样它的方法调用会被拦截，从而保证 `@Bean` 方法的调用经过 Spring 容器，使得容器可以管理这些 bean。
  
- `@Component`: 用于将类标记为 Spring 组件，交由 Spring 容器管理。Spring 会自动扫描并注册被 `@Component` 标记的类，使其成为 Spring 容器中的一个 bean。

  普通的 `@Component` 类（如 `@Service`、`@Repository` 等）在代理方面可以使用 CGLIB，但如果类实现了接口，Spring 也可以使用 JDK 动态代理。选择使用哪种代理方式通常由 Spring 配置中的 `proxyTargetClass` 属性决定。如果 `proxy-target-class` 被设置为 `true`，Spring 将使用 CGLIB 代理；如果设置为 `false` 或省略，Spring 将尽可能使用 JDK 动态代理。

总体而言，`@Configuration` 主要用于配置类，而 `@Component` 则用于普通的 Spring 组件。<font color='Chestnut Red'>**`@Configuration` 类内部的 `@Bean` 方法的调用是通过 Spring 容器来管理的，因此 `@Configuration` 类的方法调用会受到 Spring 容器的代理，而 `@Component` 注解的类则直接由 Spring 容器管理**</font>。 `@Configuration` 类的代理方式由 Spring 决定，通常是 CGLIB，但当`@Configuration` 类实现了接口并且目标对象的所有方法都在接口中声明时。可能不会使用 CGLIB，而是使用 JDK 动态代理。

例如：

```java
@Configuration
public class MyConfiguration implements MyInterface {
    @Bean
    public SomeBean someBean() {
        return new SomeBean();
    }

    @Override
    public void someMethod() {
        // Method implementation
    }
}
```

在这种情况下，由于 `MyConfiguration` 实现了接口 `MyInterface`，Spring 会优先使用 JDK 动态代理，而不是 CGLIB。这确保代理对象遵循接口的规范（接口的方法实现、接口方法签名、异常抛出）。

总的来说，`@Configuration` 类的代理方式受到类的结构和实现的接口的影响。如果该类实现了接口，且所有方法都在接口中声明，那么可能会使用 JDK 动态代理。否则，通常会使用 CGLIB。

而 `@Component` 类的代理方式则可由配置控制，可选用 CGLIB 或 JDK 动态代理。

2. <font color='Chestnut Red'>**@Repository、@Component、@Service、@Controller之间的区别与联系**</font>

`@Component`,`@Service`, `@Controller`, `@Repository`是 Spring 注解，注解后可以被 Spring 框架所扫描并注入到 Spring 容器来进行管理
`@Component`是通用注解，其他三个注解是这个注解的拓展，并且具有了特定的功能
`@Repository`注解在持久层中，具有将数据库操作抛出的原生异常翻译转化为Spring的持久层异常的功能。
`@Controller`是Spring-MVC的注解，具有将请求进行转发，重定向的功能。
`@Service`是业务逻辑层注解，这个注解只是标注该类处于业务逻辑层。
用这些注解对应用进行分层之后，就能将请求处理，业务逻辑处理，数据库操作处理分离出来，为代码解耦，也方便了以后项目的维护和开发。

3. @Autowired 和 @Resource

   > 1. **来源**：
   >    - `@Autowired` 是Spring框架提供的注解，用于进行依赖注入。
   >    - `@Resource` 是JavaEE（J2EE）规范提供的注解，也可以用于依赖注入，但它不仅可以在Spring中使用，也可以在其他JavaEE容器中使用。
   > 2. **注入方式**：
   >    - `@Autowired` 默认按照类型进行注入的，默认情况下它要求依赖对象必须存在，如果允许null值，可以设置它required属性为false。如果找到多个Bean，那么@Autowired会根据名称来查找，如果还有多个Bean，则报出异常。如果想按名称装配，也可以结合@Qualifier注解一起使用。
   >    - `@Resource` 默认按照名称进行注入，也可以指定type按照类型进行注入。
   >      - 同时指定 name 和 type，则从 Spring 上下文中找到与它们唯一匹配的 Bean 进行装配，找不到则抛出异常；
   >      - 指定 name，则从上下文中查找与名称（ID）匹配的 Bean 进行装配，找不到则抛出异常；
   >      - 指定 type，则从上下文中找到与类型匹配的唯一 Bean 进行装配，找不到或者找到多个就会抛出异常；
   >      - 既没有指定 name，也没有指定 type，则自动按 byName 方式进行装配。如果没有匹配成功，则仍按照 type 进行匹配；
   > 3. **容器支持**：
   >    - `@Autowired` 是Spring特有的注解，只能在Spring容器中使用。
   >    - `@Resource` 是JavaEE规范提供的注解，在任何JavaEE容器中都可以使用，包括Spring容器。
   > 4. **可选性**：
   >    - 在`@Autowired`中，如果找不到匹配的Bean，则注入会失败，会抛出异常。
   >    - 在`@Resource`中，如果找不到对应的Bean，则会尝试按照名称注入，如果还是找不到，则会注入失败。

# Spring

## 代理模式

### 动态代理

1. 基于接口的 JDK 动态代理

   JDK 动态代理通过反射机制生成一个实现目标接口的代理类，调用方法时通过 `InvocationHandler` 拦截处理。

2. 基于继承的 CGLib 动态代理

   CGLib 通过 ASM 字节码操作框架，生成目标类的子类作为代理类，重写父类方法实现代理逻辑。(无法代理 `final` 类或 `final` 方法)

**从特性对比**：

- JDK动态代理要求目标对象必须实现至少一个接口，因为它基于接口生成代理类。
- CGLIB动态代理不依赖于目标对象是否实现接口，可以代理没有实现接口的类，它通过继承或者代理目标对象的父类来实现代理。

**从创建代理时的性能对比**：

- JDK动态代理通常比CGLIB动态代理创建速度更快，因为它不需要生成字节码文件。
- JDK代理生成的代理类较小，占用较少的内存，而CGLIB生成的代理类通常较大，占用更多的内存。

**从调用时的性能对比**：

- 每次方法调用需通过反射（`Method.invoke`）执行，虽在 JDK 8+ 后反射性能优化显著，但仍低于直接调用。
- 通过方法重写直接调用目标方法，无需反射，性能接近原生调用。在高频调用场景下显著优于 JDK 代理。

Spring默认情况如果目标类实现了接口用JDK代理否则用CGLIB，可通过 `proxyTargetClass=true` 强制所有场景使用 CGLib。而SpringBoot默认用CGLIB，避免接口局限性带来的问题

### 静态代理

  

**核心**

- 控制反转`（IOC）`
- 依赖注入`（DI）`
- 面向切面编程`（AOP）`

## AOP

即面向切面编程，它通过在方法调用前、调用后或异常抛出时插入通知，允许开发者在核心业务逻辑之外执行横切关注点的代码。

`Spring AOP`是基于动态代理的，如果要代理的对象实现了某个接口，那么`Spring AOP`就会使用`JDK`动态代理去创建代理对象；而对于没有实现接口的对象，使用`CGlib`动态代理生成一个被代理对象的子类来作为代理。

**AOP底层实现分为两部分**：**创建AOP动态代理** 和 **调用代理**

1. 创建动态代理
   1. 启动Spring时会创建动态代理，首先通过AspectJ解析切点表达式，它会根据定义的条件匹配目标Bean的方法。
   2. 如果Bean不符合切点的条件，将跳过，否则将会通动态代理包装Bean对象：具体会根据目标对象是否实现接口来选择使用JDK动态代理或CGLIB代理。这使得AOP可以适用于各种类型的目标对象。
2. 调用阶段
   1. Spring AOP使用责任链模式来管理通知的执行顺序。通知拦截链包括前置通知、后置通知、异常通知、最终通知和环绕通知，它们按照配置的顺序形成链式结构。
   2. 通知的有序执行：责任链确保通知按照预期顺序执行。前置通知在目标方法执行前执行，后置通知在目标方法成功执行后执行，异常通知在方法抛出异常时执行，最终通知无论如何都会执行，而环绕通知包裹目标方法，允许在方法执行前后添加额外的行为。

<font color='Chestnut Red'>**应用场景**</font>

- 记录日志(调用方法后记录日志)
- 监控性能(统计方法运行时间)
- 权限控制(调用方法前校验是否有权限)
- 事务管理(调用方法前开启事务，调用方法后提交关闭事务 )
- 缓存优化(第一次调用查询数据库，将查询结果放入内存对象， 第二次调用，直接从内存对象返回，不需要查询数据库 )

<font color='Chestnut Red'>**实现 AOP 的技术，主要分为两大类：**</font>

- 静态代理：指使用 AOP 框架提供的命令进行编译，从而在编译阶段就可生成 AOP 代理类，因此也称为编译时增强； 

   - 编译时编织（特殊编译器实现）
  - 类加载时编织（特殊的类加载器实现）。
- 动态代理：运行时在内存中“临时”生成 AOP 动态代理类，因此也被称为运行时增强。 

   - JDK 动态代理 ：JDK Proxy 是 Java 语言自带的功能，通过拦截器加反射的方式实现的，只能代理实现接口的类；
   - CGLIB：CGLib 是第三方提供的工具，基于 ASM 实现的，性能比较高；无需通过接口来实现，主要是对指定的类生成一个子类，通过实现子类的方式来完成调用。

<font color='Chestnut Red'>**Spring AOP和AspectJ AOP有什么区别？**</font>

Spring AOP是属于运行时增强，而AspectJ是编译时增强。Spring AOP基于代理（Proxying），而AspectJ基于字节码操作（Bytecode Manipulation）。

Spring AOP已经集成了AspectJ，AspectJ应该算得上是Java生态系统中最完整的AOP框架了。AspectJ相比于Spring AOP功能更加强大，但是Spring AOP相对来说更简单。

如果我们的切面比较少，那么两者性能差异不大。但是，当切面太多的话，最好选择AspectJ，它比SpringAOP快很多。



**AOP的相关术语：**

- **切面（Aspect）**：共有功能的实现。如日志切面、权限切面、验签切面等。在实际开发中通常是一个存放共有功能实现的标准Java类。当Java类使用了@Aspect注解修饰时，就能被AOP容器识别为切面。

- **通知（Advice）**：切面的具体实现。就是要给目标对象织入的事情。以目标方法为参照点，根据放置的地方不同，可分为前置通知（Before）、后置通知（AfterReturning）、异常通知（AfterThrowing）、最终通知（After）与环绕通知（Around）5种。在实际开发中通常是切面类中的一个方法，具体属于哪类通知，通过方法上的注解区分。

- **连接点（JoinPoint）**：程序在运行过程中能够插入切面的地点。例如，方法调用、异常抛出等。Spring只支持方法级的连接点。一个类的所有方法前、后、抛出异常时等都是连接点。

- **切入点（Pointcut）**：用于定义通知应该切入到哪些连接点上。不同的通知通常需要切入到不同的连接点上，这种精准的匹配是由切入点的正则表达式来定义的。

  > 比如，在上面所说的连接点的基础上，来定义切入点。我们有一个类，类里有10个方法，那就产生了几十个连接点。但是我们并不想在所有方法上都织入通知，我们只想让其中的几个方法，在调用之前检验下入参是否合法，那么就用切点来定义这几个方法，让切点来筛选连接点，选中我们想要的方法。切入点就是来定义哪些类里面的哪些方法会得到通知。

- **目标对象（Target）**：那些即将切入切面的对象，也就是那些被通知的对象。这些对象专注业务本身的逻辑，所有的共有功能等待AOP容器的切入。

- **代理对象（Proxy）**：将通知应用到目标对象之后被动态创建的对象。可以简单地理解为，代理对象的功能等于目标对象本身业务逻辑加上共有功能。代理对象对于使用者而言是透明的，是程序运行过程中的产物。目标对象被织入共有功能后产生的对象。

- **织入（Weaving）**：将切面应用到目标对象从而创建一个新的代理对象的过程。这个过程可以发生在编译时、类加载时、运行时。Spring是在运行时完成织入，运行时织入通过Java语言的反射机制与动态代理机制来动态实现

## IOC

控制反转(Inversion of Control)是一种软件设计思想，它将传统上由程序代码直接操控的对象的调用权交给容器，通过容器来实现对象组件的装配和管理。所谓"控制反转"，就是对组件对象控制权的转移，从程序代码本身转移到了外部容器。

**优势：**

> - **解耦**：降低组件之间的耦合度，实现软件各层之间的解耦
> - **集中管理**：统一管理对象的创建和生命周期
> - **方便配置和管理**：便于修改配置而不必修改代码

`IoC`的主要实现方式有两种：依赖查找、依赖注入。依赖注入是一种更可取的方式。<font color='orange'>实现原理就是工厂模式加反射机制。</font>

- **工厂模式**：IoC容器本质上是一个大型工厂，负责创建、装配对象
- **反射机制**：利用Java反射动态创建对象并注入依赖

### **依赖查找、依赖注入**

- 依赖查找：主要是容器为组件提供一个回调接口和上下文环境。这样一来，组件就必须自己使用容器提供的API来查找资源和协作对象，控制反转仅体现在那些回调方法上，容器调用这些回调方法，从而应用代码获取到资源。

  > **特点**：
  >
  > - 组件主动使用容器API获取依赖
  > - 组件与容器API耦合度较高
  > - 侵入性较强，改变了组件的编码方式

  ```java 
  // 依赖查找方式
  ApplicationContext context = new ClassPathXmlApplicationContext("beans.xml");
  UserService userService = (UserService) context.getBean("userService");
  userService.doSomething();
  ```

- 依赖注入：组件不做定位查询，只提供标准的Java方法让容器去决定依赖关系。容器全权负责组件的装配，把符合依赖关系的对象通过Java Bean属性或构造方法传递给需要的对象。

  > 依赖注入`（Dependency Injection，DI）`，是组件之间依赖关系由容器在运行期决定，即由容器动态的将某个依赖关系注入到组件之中。依赖注入的目的并非为软件系统带来更多功能，而是为了提升组件重用的频率，并为系统搭建一个灵活、可扩展的平台。通过依赖注入机制，只需要通过简单的配置，而无需任何代码就可指定目标需要的资源，完成自身的业务逻辑，而不需要关心具体的资源来自何处，由谁实现。
  >
  > **`IoC` 和 `DI` 的关系**： DI是IoC的一种重要实现策略，但IoC的实现不仅限于DI，还包括依赖查找等方式。在现代框架中，DI因其低侵入性和高可测试性，已成为IoC最主流的实现方式。
  >
  > **`Spring`依赖注入方式**
  >
  > - 注解注入方式
  > - set注入方式
  > - 构造器注入方式
  >
  > 推荐使用<font color='Chestnut Red'>基于注解注入方式，配置较少，比较方便</font>。

### **IOC容器**

`IoC`容器：具有依赖注入功能的容器，可以创建对象的容器。`IoC`容器负责对象的实例化、初始化、对象的装配(依赖注入)以及<font color='Chestnut Red'>Bean生命周期的管理</font>。

依赖注入：由`IoC`容器动态地将某个对象所需要的外部资源（包括对象、资源、常量数据）注入到组件(`Controller, Service等`）之中。简单点说，就是`IoC`容器会把当前对象所需要的外部资源动态的注入给我们。

> 在Spring框架中，IoC容器实际上是一个存放各种对象的Map集合：
>
> - key：Bean的名称或ID
> - value：Bean对象
>
> **主要实现接口**：
>
> - `BeanFactory`：基础容器，提供基本的IoC功能
> - `ApplicationContext`：高级容器，扩展了BeanFactory，支持应用事件发布，集成AOP功能。
> - **ApplicationContext常见实现**：
>   - `ClassPathXmlApplicationContext`：从类路径加载配置
>   - `FileSystemXmlApplicationContext`：从文件系统加载配置
>   - `AnnotationConfigApplicationContext`：基于Java注解配置
>   - `WebApplicationContext`：为Web应用准备的专用上下文
>
> **容器初始化流程**：
>
> 1. 读取配置（XML/注解/Java配置）
> 2. 解析配置信息到BeanDefinition
> 3. 将BeanDefinition注册到BeanFactory
> 4. 实例化非懒加载的单例Bean
> 5. 触发各种Bean的初始化方法

## 循环依赖

结论：<font color='Chestnut Red'>**单例模式下属性注入的循环依赖可通过三级缓存处理循环依赖**</font>。下面两种无法解决：

1. 构造器注入的循环依赖：`Spring`处理不了，因为加入`singletonFactories`三级缓存的前提是执行了构造器来创建半成品的对象，所以构造器的循环依赖没法解决。
2. 非单例循环依赖：无法处理。因为Spring容器不缓存`prototype`类型的`bean`，使得无法提前暴露出一个创建中的`bean`。`Spring`容器在获取`prototype`类型的`bean`时，如果因为循环依赖的存在，检测到当前线程已经正在处理该`bean`时，就会抛出异常

首先，Spring单例对象的初始化大略分为三步：

1. `createBeanInstance`：实例化bean，使用构造方法创建对象，为对象分配内存。
2. `populateBean`：进行依赖注入。
3. `initializeBean`：初始化bean。

Spring为了解决单例的循环依赖问题，使用了三级缓存（三个Map）：

1. `singletonObjects`：一级缓存；存储完整的 Bean 实例，bean name --> bean instance
2. `earlySingletonObjects`：二级缓存；存储的是实例化以后，但还没有设置属性值的 Bean 实例（还没做依赖注入），bean name --> bean instance
3. `singletonFactories`： 三级缓存；存放 Bean 工厂，它主要用来生成原始 Bean 对象并且放到二级缓存里。同时，如果对象有Aop代理，则对象工厂返回代理对象。bean name --> ObjectFactory

> 1. <font color='RedOrange'>只使用一级缓存</font>：如果只是循环依赖导致的死循环的问题： 一级缓存就可以解决 ，但是在并发下会获取到不完整的Bean。
>
> <font color='orange'>方法：</font>首先一级缓存是可以解决简单对象 A 与 B 之间相互依赖的，主要是存放半成品对象，在创建 A 时，不添加属性 B，直接放入缓存里面，在给属性 B 赋值时，就需要从缓存中获取 B，如果没有，就创建 B ，在创建 B 的时候要给它的属性 A 赋值，这时候就可以从缓存中获取 A 了。
>
> <font color='orange'>存在的问题 ：</font>
>
> 早期对象和完整对象都存在于一级缓存中，如果此时来了其它线程并发获取 bean，就可能从一级缓存中获取到不完整的 bean
>
> 可以这样处理：从一级缓存获取对象处加一个互斥锁。而加互斥锁也带来了另一个问题，容器刷新完成后的普通获取 bean 的请求都需要竞争锁，如果这样处理，在高并发场景下使用 Spring 的性能一定会极低。
>
> 2. <font color='RedOrange'>只使用二级缓存</font>：可完全解决循环依赖：只是需要在实例化后就创建动态代理，不优雅也不符合spring生命周期规范。
>
> <font color='orange'>存在的问题 ：</font> 遇到 AOP 会有问题！Spring中AOP的实现是通过后置处理器方法：BeanPostProcessor机制来实现的，而后置处理器是在填充属性结束后才执行的。如果A需要被代理，由于二级缓存中的早期对象是原对象，而代理对象是在A初始化完成之后再创建的，这就导致了B对象中引用的A对象不是代理对象。
>
> 把AOP的代理对象放在二级缓存中也能解决循环依赖问题，那么这个代理对象就要在 Bean 创建完全之前就创建好，这就势必需要将BeanPostProcessor阶段提前或者侵入到填充属性的流程中，但是这样会导致维护代理对象的逻辑和getBean的逻辑过于耦合。

三级缓存的核心思想，就是把 Bean 的实例化，和 Bean 里面的依赖注入进行分离。 多级缓存并不只是为了解决循环依赖和AOP的问题，还考虑到了逻辑的分离、结构的扩展性和保证数据安全前提下的效率问题等

## Bean的作用域

1. singleton：唯一bean实例，Spring中的bean默认都是单例的。
2. prototype：每次请求都会创建一个新的bean实例。
3. request：每一次HTTP请求都会产生一个新的bean，该bean仅在当前HTTP request内有效。
4. session：每一次HTTP请求都会产生一个新的bean，该bean仅在当前HTTP session内有效。
5. global-session：全局session作用域，仅仅在基于Portlet的Web应用中才有意义，Spring5中已经没有了。Portlet是能够生成语义代码（例如HTML）片段的小型Java Web插件。它们基于Portlet容器，可以像Servlet一样处理HTTP请求。但是与Servlet不同，每个Portlet都有不同的会话。

## FactoryBean和BeanFactory

在Spring框架中，`FactoryBean`和`BeanFactory`都是用于创建和管理对象的接口，但它们的职责不同。

1. **BeanFactory**

   > `BeanFactory`：管理Bean的容器，是所有 Spring Bean 容器的顶级接口，它为 Spring 的容器定义了一套规范，并提供像 `getBean` 这样的方法从容器中获取指定的 `Bean` 实例。`BeanFactory` 在产生 `Bean` 的同时，还提供了解决 `Bean` 之间的依赖注入的能力。
   >
   > `BeanFactory`它的主要功能是通过`Bean`的名称获取`Bean`的实例对象。`BeanFactory`可以通过`XML`配置文件、注解、`Java`代码等方式来定义`Bean`，可以根据需要动态地加载和卸载`Bean`，也可以为`Bean`提供各种定制化配置。

2. **FactoryBean**

   > `FactoryBean`是一个创建`Bean`的工厂接口，主要的功能是动态生成某一个类型的 `Bean` 的实例，也就是说，我们可以自定义一个 `Bean` 并且加载到 `IOC` 容器里面。 它里面有一个重要的方法叫` getObject()`，这个方法里面就是用来实现动态构建 `Bean` 的过程。`Spring Cloud` 里面的 `OpenFeign` 组件，客户端的代理类，就是使用了 `FactoryBean` 来实现的。
   >
   > `FactoryBean`需要实现`getObject()`方法，该方法返回创建的`Bean`对象实例。它还可以实现`getObjectType()`方法，该方法返回创建的`Bean`对象的类型。这使得`FactoryBean`可以创建不同类型的`Bean`实例，并且可以根据需要动态地切换`Bean`实现。
   >
   > > 通过 getBean()方法返回的不是FactoryBean本身，而是调用FactoryBean#getObject()方法所返回的对象，相当于FactoryBean#getObject()代理了getBean()方法。如果想得到FactoryBean必须使用 '&' + beanName 的方式获取。

## Bean生命周期

<img src="https://gitee.com/qc_faith/picture/raw/master/image/202401011959001.png" alt="image-20240101195915933" style="zoom: 40%;" />

Bean的完整生命周期经历了各种方法调用，这些方法可以划分为以下几类：

- <font color='Apricot'>Bean自身的方法</font>： 包括Bean本身调用的方法及`init-method`和`destroy-method`指定的方法
- <font color='Apricot'>Bean级生命周期接口方法</font>： 这个包括了`BeanNameAware`、`BeanFactoryAware`、`ApplicationContextAware`；当然也包括`InitializingBean`和`DiposableBean`这些接口的方法（可以被`@PostConstruct`和`@PreDestroy`注解替代)
- <font color='Apricot'>容器级生命周期接口方法</font>： 这个包括了`InstantiationAwareBeanPostProcessor` 和 `BeanPostProcessor` 这两个接口实现，一般称它们的实现类为“后处理器”。
- <font color='Apricot'>工厂后处理器接口方法</font>： 这个包括了`AspectJWeavingEnabler`, `ConfigurationClassPostProcessor`, `CustomAutowireConfigurer`等等非常有用的工厂后处理器接口的方法。工厂后处理器也是容器级的。在应用上下文装配配置文件之后立即调用。

<img src="https://gitee.com/qc_faith/picture/raw/master/image/202401011924947.png" alt="img" style="zoom:67%;" />

**具体而言，流程如下**

- **实例化**：创建Bean实例（new）。
- **属性赋值**：设置属性值和依赖注入
- **初始化**：
   - 执行`xxxAware`等接口方法
     - 第一类`Aware`接口
       - 如果 `Bean` 实现了 `BeanNameAware` 接口，则 `Spring` 调用 `Bean` 的 `setBeanName()` 方法传入当前 `Bean` 的 `id` 值。
       - 如果 `Bean` 实现了 `BeanClassLoaderAware` 接口，则 `Spring` 调用 `setBeanClassLoader()` 方法传入`classLoader`的引用。
       - 如果 `Bean` 实现了 `BeanFactoryAware` 接口，则 `Spring` 调用 `setBeanFactory()` 方法传入当前工厂实例的引用。
     - 第二类`Aware`接口
       - 如果 `Bean` 实现了 `EnvironmentAware` 接口，则 `Spring` 调用 `setEnvironment()` 方法传入当前 `Environment` 实例的引用。
       - 如果 `Bean` 实现了 `EmbeddedValueResolverAware` 接口，则 `Spring` 调用 `setEmbeddedValueResolver()` 方法传入当前 `StringValueResolver` 实例的引用。
       - 如果 `Bean` 实现了 `ApplicationContextAware` 接口，则 `Spring` 调用 `setApplicationContext()` 方法传入当前 `ApplicationContext` 实例的引用。
   - 执行BeanPostProcessor的前置处理方法`postProcessBeforeInitialzation()`，此处非常重要，<font color='Magenta'>Spring 的 AOP 就是利用它实现的</font>。
   - 执行InitializingBean的afterPropertiesSet方法。(或者执行有`@PostConstruct`注解的方法)
   - 执行自定义init-method
   - 执行BeanPostProcessor的后置处理方法 `postProcessAfterInitialization()`。
- **使用**：此时，`Bean` 已经可以被应用系统使用了。
- 如果在 `<bean>` 中指定了该 Bean 的作用范围为 scope="singleton"，则将该 Bean 放入 Spring IoC 的缓存池中，将触发 Spring 对该 Bean 的生命周期管理；如果在 `<bean>` 中指定了该 Bean 的作用范围为 scope="prototype"，则将该 Bean 交给调用者，调用者管理该 Bean 的生命周期，Spring 不再管理该 Bean。
- **销毁**：
   - 执行DisposableBean的destroy方法；(或者执行有`@PreDestroy`注解的方法)
   - 执行自定义destroy-method

## Spring中的设计模式：

1. **工厂模式**
   - **抽象工厂模式（Abstract Factory）**
     **`BeanFactory` 和 `ApplicationContext`** 是 Spring 容器的基础接口，属于抽象工厂模式。它们定义了一组创建 Bean 的通用方法，具体实现（如 `DefaultListableBeanFactory`）负责生成不同类型的 Bean 实例，体现了抽象工厂中“生产一系列相关对象”的思想。
   - **工厂方法模式（Factory Method）**
     **`FactoryBean`** 接口是工厂方法模式的典型应用。通过实现 `FactoryBean`，用户可以自定义复杂对象的创建逻辑，由 Spring 容器调用 `getObject()` 方法生成目标 Bean。
2. **单例模式（Singleton）**
   - Spring 默认以单例作用域（`singleton`）管理 Bean，确保容器中每个 Bean 对应唯一实例，以提高性能和资源复用。
3. **代理模式（Proxy）**
   - Spring AOP 通过 **JDK 动态代理**（基于接口）和 **CGLIB 代理**（基于继承）实现切面功能，动态生成代理对象以增强目标方法。
4. **观察者模式（Observer）**
   - **事件驱动模型**：通过 `ApplicationEvent` 和 `ApplicationListener` 实现松耦合的事件发布与监听机制。例如，容器启动事件（`ContextRefreshedEvent`）可被监听并触发响应逻辑。
5. **模板方法模式（Template Method）**
   - **`JdbcTemplate`、`RestTemplate`** 等工具类封装了固定流程（如连接获取/释放），同时将可变部分（如 SQL 执行）以回调形式（`RowMapper`、`ResultSetExtractor`）交给用户扩展，符合模板方法模式。
6. **策略模式（Strategy）**
   - **`@ComponentScan` 中的过滤器**：通过 `includeFilters` 和 `excludeFilters` 定义组件扫描策略，允许灵活切换过滤规则。依赖注入时选择不同实现类（如 `@Qualifier`）也体现了策略模式。
7. **装饰器模式（Decorator）**
   - **`HttpServletRequestWrapper`** 等包装类：在 Spring Web 中，装饰器模式用于动态增强请求/响应对象的功能，而非直接修改原始对象。
8. **责任链模式（Chain of Responsibility）**
   - **AOP 的通知链**：如 `MethodInterceptor` 形成拦截器链，多个通知按顺序执行，责任链模式确保每个拦截器独立处理调用。
   - **Spring Security 的过滤器链**：请求依次通过多个安全检查过滤器。
9. **原型模式（Prototype）**
   - 当 Bean 的作用域为 `prototype` 时，每次从容器获取的都是新实例，这是原型模式的应用。
10. **建造者模式（Builder）**
   - **`BeanDefinitionBuilder`**：用于逐步构造复杂的 `BeanDefinition` 对象，通过链式调用配置属性，解耦对象构造过程与表示形式。
11. **适配器模式（Adapter）**
    - **`HandlerAdapter`** 在 Spring MVC 中将不同类型的 Controller（如 `@Controller`、`HttpRequestHandler`）适配到统一的处理接口，使得多样化请求处理器能够以一致的方式工作。


## 面试题

### Spring中的Bean是线程安全的吗？

Spring容器只负责创建和管理Bean，并未提供Bean的线程安全策略。因此，Spring容器中的Bean并不具备线程安全的特性。Spring中的Bean又可分为多例bean和单例bean，单例bean又可分为无状态的bean和有状态的bean。多例bean和无状态的bean是不需要考虑线程安全的。只有有状态的单例bean是需要考虑线程安全的，可使用ThreadLocal来管理成员变量，使之线程隔离，以达到线程安全的目的。

> 1、Spring中Bean从哪里来的？
> 在Spring容器中，除了很多Spring内置的Bean以外，其他的Bean都是我们自己通过Spring配置来声明的，然后，由Spring容器统一加载。我们在Spring声明配置中通常会配置以下内容，如：class（全类名）、id（也就是Bean的唯一标识）、 scope（作用域）以及lazy-init（是否延时加载）等。之后，Spring容器根据配置内容使用对应的策略来创建Bean的实例。因此，Spring容器中的Bean其实都是根据我们自己写的类来创建的实例。因此，Spring中的Bean是否线程安全，跟Spring容器无关，只是交由Spring容器托管而已。
> 那么，在Spring容器中，什么样的Bean会存在线程安全问题呢？回答，这个问题之前我们得先回顾一下Spring Bean的作用域。在Spring定义的作用域中，其中有 prototype（ 多例Bean ）和 singleton （ 单例Bean）。那么，定义为 prototype 的Bean，是在每次 getBean 的时候都会创建一个新的对象。定义为 singleton 的Bean，在Spring容器中只会存在一个全局共享的实例。基于对以上Spring Bean作用域的理解，下面，我们来分析一下在Spring容器中，什么样的Bean会存在线程安全问题。
>
> 2、Spring中什么样的Bean存在线程安全问题？
> 我们已经知道，多例Bean每次都会新创建新实例，也就是说线程之间不存在Bean共享的问题。因此，多例Bean是不存在线程安全问题的。
> 而单例Bean是所有线程共享一个实例，因此，就可能会存在线程安全问题。但是单例Bean又分为无状态Bean和有状态Bean。在多线程操作中只会对Bean的成员变量进行查询操作，不会修改成员变量的值，这样的Bean称之为无状态Bean。所以，可想而知，无状态的单例Bean是不存在线程安全问题的。但是，在多线程操作中如果需要对Bean中的成员变量进行数据更新操作，这样的Bean称之为有状态Bean，所以，有状态的单例Bean就可能存在线程安全问题。
> 所以，最终我们得出结论，在Spring中，只有有状态的单例Bean才会存在线程安全问题。我们在使用Spring的过程中，经常会使用到有状态的单例Bean，如果真正遇到了线程安全问题，我们又该如何处理呢？
>
> 3、如何处理Spring Bean的线程安全问题？
> 处理有状态单例Bean的线程安全问题有以下三种方法：
> 1、将Bean的作用域由 “singleton” 单例 改为 “prototype” 多例。
> 2、在Bean对象中避免定义可变的成员变量，当然，这样做不太现实，就当我没说。
> 3、在类中定义 ThreadLocal 的成员变量，并将需要的可变成员变量保存在 ThreadLocal 中，ThreadLocal 本身就具备线程隔离的特性，这就相当于为每个线程提供了一个独立的变量副本，每个线程只需要操作自己的线程副本变量，从而解决线程安全问题。

[Spring中的Bean是线程安全的吗？ - 简书 (jianshu.com)](https://www.jianshu.com/p/be1db76330d8)

[Spring中的Bean是线程安全的吗？ - SegmentFault 思否](https://segmentfault.com/a/1190000041686441?sort=newest)
