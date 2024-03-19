# AbstraceApplicationContext refresh 方法及扩展

<font color='Chestnut Red'>**一、prepareRefresh()**</font>

刷新前的准备 加载 PropertySource 可扩展点 `propertySource`
1、继承/实现 Environment, 并在 customizePropertySources 方法 加入自己想配置的 propertySource
2、继承 AnnotationConfigServletWebServerApplicationContext 并实现 initPropertySources， 添加自己的 propertySource
3、在 ApplicationContext refresh 前获取 enviroment，并添加 propertySource

<font color='Chestnut Red'>**二、obtainFreshBeanFactory()**</font>

获取 beanFactory，没有扩展点

<font color='Chestnut Red'>**三、prepareBeanFactory(beanFactory)**</font>

准备beanFacoty的配置
1、指定classLoader
2、指定SpEL
3、指定属性编辑注册器ProperytEditorRegistrar
4、指定预先beanPostProcessor， 如各种aware织入处理，包括ApplicationContextAwareProcessor(用于处理自定义bean织入ApplicationContext)
ApplicationListenerDetector(用于提早发现bean定义的监听器)
5、注册特殊依赖，和排除特殊依赖
6、检查是否有类加载织入
7、注册特殊bean，如environment, jvm配置参数，系统配置参数

通过以上生命以及代码的声明方式，可以扩展的有
1、在refresh前，获取beanFactory，并添加beanPostProcessor，这样添加的beanProstProcessor比一般的beanProstProcessor能早处理

<font color='Chestnut Red'>**四、postProcessBeanFactory(beanFactory)**</font>

处理beanFactory, 默认什么都没触发

<font color='Chestnut Red'>**五、invokeBeanFactoryPostProcessors(beanFactory)**</font>

- 首先初始化并按顺序批次触发所有BeanDefinitionRegistryPostProcessor

  (BeanDefinitionRegistryPostProcessor是BeanFactoryPostProcessor的子接口)

  - 这里包括处理@Configuration/@Component/@Import/@ImportResource/@ComponentScan的BeanDefinition加载注册/解析，并一起处理一下步骤
  - 包括类启动类上注释的其他注解，以及注解里引入的注解如@Import,@ComponentScan等
    - 包括类/注解类上声明的@ImportSelector(如spring-boot的执行**AutoConfigrationImportSelector**类的getAutoConfigurationEntry方法加载所有spring.factories的EnableAutoConfiguration)beanDefinition加载注册
      这里是给BeanFacotyPostProcessor触发的机会的，当然也可以实现BeanFacotyPostProcessor来干别的事情
      比如PropertyPlaceholderConfiger实现了BeanFactoryPostProcessor处理占位符

- BeanDefinitionRegistryPostProcessor会被多次执行，一次是参数传递，一次是查询，还有一次是上一次查询附加查询出来的BeanDefinitionRegistryPostProcessor

- 然后初始化并按顺序批次**触发**所有**BeanFactoryPostProcessor**， 一样的会被执行多次

<font color='Chestnut Red'>**六、registerBeanPostProcessors(beanFactory)**</font>

注册beanPostProcessor, 即所有BeanPostProcessor的扩展

<font color='Chestnut Red'>**七、initMessageSource ()**</font>

初始化国际化相关

<font color='Chestnut Red'>**八、initApplicationEventMulticaster()**</font>

初始化事件分发相关

<font color='Chestnut Red'>**九、onRefresh()**</font>

默认空实现，ApplicationContext的子类可以实现该方法，做自己特殊bean的初始化等。

<font color='Chestnut Red'>**十、registerListeners()**</font>

注册spring事件监听器，并广播可能的事件。 继承ApplicationListener即可实现自己的监听。
因为走到这一步，应用已经刷新，**即无法实现**应用启动前，启动中的事件监听。
spring-boot封装SpringApplicationRunListener, 并封装SpringAppliction的引导启动流程，能在Spring的启动过程中发出各个事件, 详情请看[spring-boot启动](https://www.jianshu.com/p/301582a9fade)

<font color='Chestnut Red'>**十一、finishBeanFactoryInitialization(beanFactory)**</font>

初始化非延迟实例化的单例

<font color='Chestnut Red'>**十二、finishRefresh()**</font>

结束刷新, 对bean生命周期监视。发送刷新完毕事件. 监视存活的bean(MBean)如果有必要

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



## 启动流程

**1. 创建并启动计时监控类**

计时器是为了监控并记录` Spring Boot `应用启动的时间的，它会记录当前任务的名称，然后开启计时器。

**2. 声明应用上下文对象和异常报告集合**

声明了应用上下文对象和一个异常报告的 `ArrayList` 集合。

**3. 设置系统属性 headless 的值**

设置` Java.awt.headless = true`，其中` awt（Abstract Window Toolkit）`的含义是抽象窗口工具集。设置为 `true` 表示运行一个 `headless` 服务器，可以用它来作一些简单的图像处理。

**4. 创建所有 Spring 运行监听器并发布应用启动事件**

获取配置的监听器名称并实例化所有的类。

**5. 初始化默认应用的参数类**

声明并创建一个应用参数对象。

**6. 准备环境**

创建配置并且绑定环境（通过` property sources `和 `profiles` 等配置文件）。

**7. 创建 Banner 的打印类**

`SpringBoot` 启动时会打印 `Banner` 图片

**8. 创建应用上下文**

创建 `ApplicationContext` 上下文对象。

**9. 实例化异常报告器**

执行 `getSpringFactoriesInstances ()` 方法获取异常类的名称，并通过反射实例化。

**10. 准备应用上下文**

把上面步骤已创建好的对象，设置到 `prepareContext` 中准备上下文。

**11. 刷新应用上下文**

解析配置文件，加载 `bean` 对象，并启动内置的 `web` 容器等等。

**12. 事件处理**

一些自定义的后置处理操作。

**13. 停止计时器监控类**

停止此过程第一步中的程序计时器，并统计任务的执行信息。

**14. 输出日志信息**

把相关的记录信息，如类名、时间等信息进行控制台输出。

**15. 发布应用上下文启动完成事件**

触发所有 `SpringApplicationRunListener` 监听器的 `started` 事件方法。

**16. 执行所有 Runner 运行器**

执行所有的 `ApplicationRunner` 和 `CommandLineRunner` 运行器。

**17. 发布应用上下文就绪事件**

触发所有的 `SpringApplicationRunListener` 监听器的 `running` 事件。

**18. 返回应用上下文对象**

至此，`SpringBoot` 启动完成。



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

**配置文件位置**

一般来说，默认的配置文件`application.properties`或者`application.yml`文件放在目录

**properties格式**

`properties`配置文件属于比较常见的一种了，定义也比较简单，形如 `key=value`

**yml格式**

`yml`格式的配置文件是以缩进来表示分层，`kv`之间用冒号来分割

**配置读取**

1. `Environment` 读取
   所有的配置信息，都会加载到`Environment`实体中，因此可以通过这个对象来获取系统的配置，通过这种方式不仅可以获取`application.yml`配置信息，还可以获取更多的系统信息

2. `@Value` 注解方式
   `@Value`注解可以将配置信息注入到`Bean`的属性，也是比较常见的使用方式，但有几点需要额外注意

   如果配置信息不存在会怎样？
   配置冲突了会怎样（即多个配置文件中有同一个key时）？
   使用方式如下，主要是通过` ${}`，大括号内为配置的`Key`；如果配置不存在时，给一个默认值时，可以用冒号分割，后面为具体的值

3. 对象映射方式

   - 通过注解`ConfigurationProperties`来制定配置的前缀
   - 通过`Bean`的属性名，补上前缀，来完整定位配置信息的`Key`，并获取`Value`赋值给这个`Bean`

##常用的stater。如何理解stater

`spring-boot-starter-web `

引入了这个`maven`依赖，基本就满足了日常的`web`接口开发，而不使用`springboot`时，需要引入`spring-web、spring-webmvc、spring-aop`等等来支持项目开发。

`starter`可以当成是一个`maven`依赖组，引入这个组名就引入了相关的依赖和一些初始化的配置。

![img](https://gitee.com/qc_faith/picture/raw/master/image/202211051829363.png)



##启动时运行特定的代码

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

## 自动配置原理

Spring Boot的自动装配是通过`@EnableAutoConfiguration`注解实现的。Spring Boot通过条件化配置（Conditional Configuration）和SPI（Service Provider Interface）机制，实现了一种基于约定优于配置的自动化配置策略。

1. **@EnableAutoConfiguration注解：** Spring Boot应用中通常会在启动类上使用`@SpringBootApplication`注解，而`@SpringBootApplication`注解本身包含了`@EnableAutoConfiguration`注解。`@EnableAutoConfiguration`注解是Spring Boot的关键注解之一，它启用了Spring Boot的自动配置机制。
2. **@Conditional注解：** Spring Boot中的很多自动配置类都使用了条件注解（`@Conditional`）进行条件判断。条件注解允许根据满足一定条件的情况来决定是否应用某个配置。
3. **spring.factories文件：** Spring Boot的自动配置信息通常是通过`META-INF/spring.factories`文件中的配置来定义的。这个文件位于项目的classpath下，其中包含了各种自动配置类的配置信息。每个自动配置类都会在`spring.factories`中定义一个`EnableAutoConfiguration`的键，指向该自动配置类的类路径。
4. **SPI机制：** Spring Boot利用了Java的SPI（Service Provider Interface）机制。在`spring.factories`文件中，配置了各种自动配置类对应的`EnableAutoConfiguration`实现类。Spring Boot在启动时，会通过SPI机制加载这些实现类，并进行相应的自动配置。
5. **自动配置类：** 自动配置类通常包含了一些`@Configuration`注解和`@Conditional`注解，以及各种`@Bean`定义。这些自动配置类的目的是根据条件判断是否需要自动配置一些Bean，以及如何配置这些Bean。

#事务

## @Transactional注意事项

Java中异常的基类为Throwable，有两个子类Exception与Errors。RuntimeException就是Exception的子类，只有RuntimeException才会进行回滚；

- `Spring`默认情况会对`(RuntimeException)`及其子类来进行回滚，在遇见`Exception`或者`Checked`异常(`ClassNotFoundException/NoSuchMetodException`)的时候则不会进行回滚操作。要求我们在自定义异常的时候，让自定义的异常继承自RuntimeException，这样抛出的时候才会被Spring默认的事务处理准确处理。
- `@Transactional`注解只能被应用到`public`方法上，这是由`Spring Aop`本质决定的。

##@Transactional注解属性含义

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

##事务失效

在`Spring`中，事务有两种实现方式

编程式事务管理：编程式事务管理使用`TransactionTemplate`或者直接使用底层的`PlatformTransactionManager`。对于编程式事务管理，`Spring`推荐使用`TransactionTemplate`。

声明式事务管理： 建立在`AOP`之上的。其本质是对方法前后进行拦截，然后在目标方法开始之前创建或者加入一个事务，在执行完目标方法之后根据执行情况提交或者回滚事务。

声明式事务管理不需要入侵代码，通过`@Transactional`就可以进行事务操作，更快捷而且简单。推荐使用

### 失效

<img src="https://gitee.com/qc_faith/picture/raw/master/image/202305301602537.png" alt="image-20230530160230417" style="zoom:33%;" />

1. `Transactional`注解标注方法修饰符为非`public`时，`@Transactional`注解将会不起作用。

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

   JDK 动态代理是利用反射机制生成一个实现代理接口的匿名类，在调用具体方法前调用InvocationHandler 来处理。

2. 基于继承的 CGLib 动态代理

   CGLib 动态代理是利用asm开源包，对代理对象类的class文件加载进来，通过修改其字节码生成子类来处理。

### 静态代理

  

**核心**

- 控制反转`（IOC）`
- 依赖注入`（DI）`
- 面向切面编程`（AOP）`

## AOP

即面向切面编程，能够将那些与业务无关，却为业务模块所共同调用的逻辑或责任（例如事务处理、日志管理、权限控制等）封装起来，便于减少系统的重复代码，降低模块间的耦合度，并有利于未来的可扩展性和可维护性。

`Spring AOP`是基于动态代理的，如果要代理的对象实现了某个接口，那么`Spring AOP`就会使用`JDK`动态代理去创建代理对象；而对于没有实现接口的对象，使用`CGlib`动态代理生成一个被代理对象的子类来作为代理。

<font color='Chestnut Red'>**`JDK`动态代理与`CGlib`动态代理的区别：**</font>

> `JDK`是基于反射机制，生成一个实现代理接口的匿名类，然后重写方法，实现方法的增强；它生成类的速度很快，但是运行时因为是基于反射，调用后续的类操作会很慢；而且他是只能针对接口编程的.
>
> `CGLIB`是基于继承机制，继承被代理类，所以方法不要声明为`final`，然后重写父类方法达到增强了类的作用。它底层是基于`asm`第三方框架，是对代理对象类的`class`文件加载进来，通过修改其字节码生成子类来处理；生成类的速度慢，但是后续执行类的操作时候很快。可以针对类和接口.
>
> 因为`jdk`是基于反射，`CGLIB`是基于字节码.所以性能上会有差异.

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

   - JDK 动态代理 
    - JDK Proxy 是 Java 语言自带的功能，无需通过加载第三方类实现；
    - Java 对 JDK Proxy 提供了稳定的支持，并且会持续的升级和更新，Java 8 版本中的 JDK Proxy 性能相比于之前版本提升了很多；
    - JDK Proxy 是通过拦截器加反射的方式实现的；
    - JDK Proxy 只能代理实现接口的类；
    - JDK Proxy 实现和调用起来比较简单；
  - CGLIB 
    - CGLib 是第三方提供的工具，基于 ASM 实现的，性能比较高；
    - CGLib 无需通过接口来实现，它是针对类实现代理，主要是对指定的类生成一个子类，它是通过实现子类的方式来完成调用的。

<font color='Chestnut Red'>**Spring AOP和AspectJ AOP有什么区别？**</font>

Spring AOP是属于运行时增强，而AspectJ是编译时增强。Spring AOP基于代理（Proxying），而AspectJ基于字节码操作（Bytecode Manipulation）。

Spring AOP已经集成了AspectJ，AspectJ应该算得上是Java生态系统中最完整的AOP框架了。AspectJ相比于Spring AOP功能更加强大，但是Spring AOP相对来说更简单。

如果我们的切面比较少，那么两者性能差异不大。但是，当切面太多的话，最好选择AspectJ，它比SpringAOP快很多。



**AOP的相关术语：**

**切面（Aspect）**：共有功能的实现。如日志切面、权限切面、验签切面等。在实际开发中通常是一个存放共有功能实现的标准Java类。当Java类使用了@Aspect注解修饰时，就能被AOP容器识别为切面。

**通知（Advice）**：切面的具体实现。就是要给目标对象织入的事情。以目标方法为参照点，根据放置的地方不同，可分为前置通知（Before）、后置通知（AfterReturning）、异常通知（AfterThrowing）、最终通知（After）与环绕通知（Around）5种。在实际开发中通常是切面类中的一个方法，具体属于哪类通知，通过方法上的注解区分。

**连接点（JoinPoint）**：程序在运行过程中能够插入切面的地点。例如，方法调用、异常抛出等。Spring只支持方法级的连接点。一个类的所有方法前、后、抛出异常时等都是连接点。

**切入点（Pointcut）**：用于定义通知应该切入到哪些连接点上。不同的通知通常需要切入到不同的连接点上，这种精准的匹配是由切入点的正则表达式来定义的。

比如，在上面所说的连接点的基础上，来定义切入点。我们有一个类，类里有10个方法，那就产生了几十个连接点。但是我们并不想在所有方法上都织入通知，我们只想让其中的几个方法，在调用之前检验下入参是否合法，那么就用切点来定义这几个方法，让切点来筛选连接点，选中我们想要的方法。切入点就是来定义哪些类里面的哪些方法会得到通知。

**目标对象（Target）**：那些即将切入切面的对象，也就是那些被通知的对象。这些对象专注业务本身的逻辑，所有的共有功能等待AOP容器的切入。

**代理对象（Proxy）**：将通知应用到目标对象之后被动态创建的对象。可以简单地理解为，代理对象的功能等于目标对象本身业务逻辑加上共有功能。代理对象对于使用者而言是透明的，是程序运行过程中的产物。目标对象被织入共有功能后产生的对象。

**织入（Weaving）**：将切面应用到目标对象从而创建一个新的代理对象的过程。这个过程可以发生在编译时、类加载时、运行时。Spring是在运行时完成织入，运行时织入通过Java语言的反射机制与动态代理机制来动态实现

## IOC

控制反转是一种设计思想，就是将原本在程序中手动创建对象的控制权，交给`IOC`容器来管理，并由`IOC`容器完成对象的注入。这样可以很大程度上简化应用的开发，把应用从复杂的依赖关系中解放出来。`IOC`容器就像是一个工厂一样，当我们需要创建一个对象的时候，只需要配置好配置文件/注解即可，完全不用考虑对象是如何被创建出来的。

`IoC`的主要实现方式有两种：依赖查找、依赖注入。依赖注入是一种更可取的方式。<font color='orange'>实现原理就是工厂模式加反射机制。</font>

**区别**

依赖查找，主要是容器为组件提供一个回调接口和上下文环境。这样一来，组件就必须自己使用容器提供的API来查找资源和协作对象，控制反转仅体现在那些回调方法上，容器调用这些回调方法，从而应用代码获取到资源。

依赖注入，组件不做定位查询，只提供标准的Java方法让容器去决定依赖关系。容器全权负责组件的装配，把符合依赖关系的对象通过Java Bean属性或构造方法传递给需要的对象。

**IOC容器**

`IoC`容器：具有依赖注入功能的容器，可以创建对象的容器。`IoC`容器负责实例化、定位、配置应用程序中的对象并建立这些对象之间的依赖。

依赖注入：由`IoC`容器动态地将某个对象所需要的外部资源（包括对象、资源、常量数据）注入到组件(`Controller, Service等`）之中。简单点说，就是`IoC`容器会把当前对象所需要的外部资源动态的注入给我们。

## 循环依赖

结论：<font color='Chestnut Red'>**单例模式下属性注入的循环依赖可通过三级缓存处理循环依赖**</font>。下面两种无法解决：

1. 构造器注入的循环依赖：`Spring`处理不了，因为加入`singletonFactories`三级缓存的前提是执行了构造器来创建半成品的对象，所以构造器的循环依赖没法解决。
2. 非单例循环依赖：无法处理。因为Spring容器不缓存`prototype`类型的`bean`，使得无法提前暴露出一个创建中的`bean`。`Spring`容器在获取`prototype`类型的`bean`时，如果因为循环依赖的存在，检测到当前线程已经正在处理该`bean`时，就会抛出异常

首先，Spring单例对象的初始化大略分为三步：

1. `createBeanInstance`：实例化bean，使用构造方法创建对象，为对象分配内存。
2. `populateBean`：进行依赖注入。
3. `initializeBean`：初始化bean。

Spring为了解决单例的循环依赖问题，使用了三级缓存：

`singletonObjects`：一级缓存；存储完整的 Bean 实例，bean name --> bean instance

`earlySingletonObjects`：二级缓存；存储的是实例化以后，但还没有设置属性值的 Bean 实例（还没做依赖注入），bean name --> bean instance

`singletonFactories`： 三级缓存；存放 Bean 工厂，它主要用来生成原始 Bean 对象并且放到二级缓存里。bean name --> ObjectFactory

> 1. <font color='RedOrange'>只使用一级缓存</font>
>
> <font color='orange'>方法：</font>首先一级缓存是可以解决简单对象 A 与 B 之间相互依赖的，主要是存放半成品对象，在创建 A 时，不添加属性 B，直接放入缓存里面，在给属性 B 赋值时，就需要从缓存中获取 B，如果没有，就创建 B ，在创建 B 的时候要给它的属性 A 赋值，这时候就可以从缓存中获取 A 了。
>
> <font color='orange'>存在的问题 ：</font>
>
> 早期对象和完整对象都存在于一级缓存中，如果此时来了其它线程并发获取 bean，就可能从一级缓存中获取到不完整的 bean
>
> 可以这样处理：从一级缓存获取对象处加一个互斥锁。而加互斥锁也带来了另一个问题，容器刷新完成后的普通获取 bean 的请求都需要竞争锁，如果这样处理，在高并发场景下使用 Spring 的性能一定会极低。
>
> 2. <font color='RedOrange'>只使用二级缓存</font>
>
> <font color='orange'>存在的问题 ：</font> 遇到 AOP 会有问题！Spring中AOP的实现是通过后置处理器方法：BeanPostProcessor机制来实现的，而后置处理器是在填充属性结束后才执行的。如果A需要被代理，由于二级缓存中的早期对象是原对象，而代理对象是在A初始化完成之后再创建的，这就导致了B对象中引用的A对象不是代理对象。
>
> 把AOP的代理对象放在二级缓存中也能解决循环依赖问题，那么这个代理对象就要在 Bean 创建完全之前就创建好，这就势必需要将BeanPostProcessor阶段提前或者侵入到填充属性的流程中，但是这样会导致维护代理对象的逻辑和getBean的逻辑过于耦合。

三级缓存的核心思想，就是把 Bean 的实例化，和 Bean 里面的依赖注入进行分离。 多级缓存并不只是为了解决循环依赖和AOP的问题，还考虑到了逻辑的分离、结构的扩展性和保证数据安全前提下的效率问题等

##依赖注入（DI）

依赖注入`（Dependency Injection，DI）`，是组件之间依赖关系由容器在运行期决定，即由容器动态的将某个依赖关系注入到组件之中。依赖注入的目的并非为软件系统带来更多功能，而是为了提升组件重用的频率，并为系统搭建一个灵活、可扩展的平台。通过依赖注入机制，只需要通过简单的配置，而无需任何代码就可指定目标需要的资源，完成自身的业务逻辑，而不需要关心具体的资源来自何处，由谁实现。

**`IoC` 和 `DI` 的关系**： `DI` 是实现 `IoC` 的方法和手段。

**`Spring`依赖注入方式**

- 注解注入方式
- set注入方式
- 构造器注入方式

推荐使用<font color='Chestnut Red'>基于注解注入方式，配置较少，比较方便</font>。

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

- 如果 `BeanFactoryPostProcessor` 和 `Bean` 关联, 则调用`postProcessBeanFactory`方法.(即<font color='RedOrange'>首先尝试从Bean工厂中获取Bean</font>)

- 如果 `InstantiationAwareBeanPostProcessor` 和 `Bean` 关联，则调用`postProcessBeforeInstantiation`方法

- 根据配置情况调用 `Bean` 构造方法<font color='RedOrange'>实例化 Bean</font>。

- 利用依赖注入完成 `Bean` 中所有<font color='RedOrange'>属性值的配置注入</font>。

- 如果 `InstantiationAwareBeanPostProcessor` 和 `Bean` 关联，则调用`postProcessAfterInstantiation`方法和`postProcessProperties`

- 调用`xxxAware`接口

   (上图只是给了几个例子) 

  - 第一类`Aware`接口
    - 如果 `Bean` 实现了 `BeanNameAware` 接口，则 `Spring` 调用 `Bean` 的 `setBeanName()` 方法传入当前 `Bean` 的 `id` 值。
    - 如果 `Bean` 实现了 `BeanClassLoaderAware` 接口，则 `Spring` 调用 `setBeanClassLoader()` 方法传入`classLoader`的引用。
    - 如果 `Bean` 实现了 `BeanFactoryAware` 接口，则 `Spring` 调用 `setBeanFactory()` 方法传入当前工厂实例的引用。
  - 第二类`Aware`接口
    - 如果 `Bean` 实现了 `EnvironmentAware` 接口，则 `Spring` 调用 `setEnvironment()` 方法传入当前 `Environment` 实例的引用。
    - 如果 `Bean` 实现了 `EmbeddedValueResolverAware` 接口，则 `Spring` 调用 `setEmbeddedValueResolver()` 方法传入当前 `StringValueResolver` 实例的引用。
    - 如果 `Bean` 实现了 `ApplicationContextAware` 接口，则 `Spring` 调用 `setApplicationContext()` 方法传入当前 `ApplicationContext` 实例的引用。
    - ...

- 如果 `BeanPostProcessor` 和 `Bean` 关联，则 `Spring` 将调用该接口的预初始化方法 `postProcessBeforeInitialzation()` 对 `Bean` 进行加工操作，此处非常重要，<font color='Magenta'>Spring 的 AOP 就是利用它实现的</font>。

- 如果 `Bean` 实现了 `InitializingBean` 接口，则 `Spring` 将调用 `afterPropertiesSet()` 方法。(或者有执行`@PostConstruct`注解的方法)

- 如果在配置文件中通过 <font color='RedOrange'>init-method</font> 属性指定了初始化方法，则调用该初始化方法。

- 如果 `BeanPostProcessor` 和 `Bean` 关联，则 `Spring` 将调用该接口的初始化方法 `postProcessAfterInitialization()`。此时，`Bean` 已经可以被应用系统使用了。

- 如果在 `<bean>` 中指定了该 Bean 的作用范围为 scope="singleton"，则将该 Bean 放入 Spring IoC 的缓存池中，将触发 Spring 对该 Bean 的生命周期管理；如果在 `<bean>` 中指定了该 Bean 的作用范围为 scope="prototype"，则将该 Bean 交给调用者，调用者管理该 Bean 的生命周期，Spring 不再管理该 Bean。

- 如果 Bean 实现了 `DisposableBean` 接口，则 `Spring` 会调用 `destory()` 方法将 `Spring` 中的 `Bean` 销毁；(或者有执行`@PreDestroy`注解的方法)

- 如果在配置文件中通过 <font color='RedOrange'>destory-method</font> 属性指定了 `Bean` 的销毁方法，则 `Spring` 将调用该方法对 `Bean` 进行销毁。

## 面试题

###Spring中的Bean是线程安全的吗？

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
