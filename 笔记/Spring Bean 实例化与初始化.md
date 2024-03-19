参考：https://blog.csdn.net/qq_15037231/article/details/105938673

本次通过源码介绍[ApplicationContext](https://so.csdn.net/so/search?q=ApplicationContext&spm=1001.2101.3001.7020)容器初始化流程，主要介绍容器内bean的实例化和初始化过程。`ApplicationContext`是`Spring`推出的先进Ioc容器，它继承了旧版本Ioc容器`BeanFactory`，并进一步扩展了容器的功能，增加了`bean`的自动识别、自动初始化等功能，同时引入了`BeanFactoryPostProcessor`、`BeanPostProcessor`等逻辑处理组件，目前我们使用的`Spring`项目大部分都是基于`ApplicationContext`容器的。`ApplicationContext`容器在启动阶段便会调用所有`bean`的构建方法`getBean()`，所以当项目启动好后，容器内所有对象都已被构建好了。

<font color='Chestnut Red'>**我个人习惯将spring容器启动分为三个阶段：（1）容器初始化阶段、（2）Bean实例化(instantiation)阶段、（3）Bean初始化(initialization)阶段。**</font>

1. **容器初始化阶段**：这个阶段主要是通过某些工具类加载`Configuration MetaData`，并将相关信息解析成`BeanDefinition`注册到`BeanDefinitionRegistry`中，即对象管理信息的收集。同时也会进行各种处理器的注册、上下文的初始化、事件广播的初始化等等准备工作。
2. **Bean实例化(instantiation)阶段**：这个阶段主要是`Bean`的实例化，其实就是`Bean`对象的构造。不过此时构造出的对象，他们内部的依赖对象仍然没有注入，只是通过反射(`或Cglib`)生成了具体对象的实例(执行构造函数)，有点类似于我们手动`new`对象，`new`出的对象已经执行过了构造函数，并且内部的基本数据也已经准备好了，但如果内部还有其他对象的依赖，就需要后续的流程去主动注入。
3. **Bean初始化(initialization)阶段**：这个阶段主要是对实例化好后的Bean进行依赖注入的过程。同时还会执行用户自定义的一些初始化方法，注册`Bean`的销毁方法、缓存初始化好的Bean等。

接下来我们主要围绕这三个阶段通过`Spring`源码来看容器初始化的具体过程。

# 一.容器初始化阶段

`ApplicationContext`在容器启动时会自动帮我们构建好所有的`Bean`，所以当我们项目启动好后，容器内所有对象都已经构建好了。其主要的逻辑都在`refresh()`函数中，所以我们也从这个函数开始看，下面是`refresh()`的主体逻辑。
<img src="https://gitee.com/qc_faith/picture/raw/master/image/202212291649578.png" alt="在这里插入图片描述" style="zoom:50%;" />
我们简单介绍下几个重要函数：

1. **prepareRefresh()**：
   初始化上下文环境，对系统的环境变量或者系统属性进行准备和校验，如环境变量中必须设置某个值才能运行，否则不能运行，这个时候可以在这里加这个校验，重写AnnotationConfigServletWebServerApplicationContext 类的 initPropertySources`方法就好了。
2. **postProcessBeanFactory()、invokeBeanFactoryPostProcessors()**：
   `Spring`提供了一种叫做`BeanFactoryPostProcessor`的容器扩展机制，这种机制可以让我们在`Bean`实例化前，对注册到容器的`BeanDefinition`所保存的信息进行修改。上面两个函数就主要是`BeanFactoryPostProcessor`的相关操作。值得一提的是我们经常会在`properties`文件中配置一些属性值，然后通过`${}`的占位符去替换`xml`中的一些变量，这个过程就是`BeanFactoryPostProcessor`帮我实现的。
3. **registerBeanPostProcessors**：
   注册`BeanPostProcessor`。它和`BeanFactoryPostProcessor`相对应，`BeanFactoryPostProcessor`主要是对`BeanDefinition`进行相关操作，而`BeanPostProcessor`主要是插手`Bean`的构建过程。`BeanPostProcessor`是个接口，它有很多实现类。
4. **finishBeanFactoryInitialization**：
   初始化剩下的单例`Bean`(非延迟加载的)。上面的函数中构建了各种系统使用的`Bean`，而剩下的`Bean`都会在`finishBeanFactoryInitialization`中构建，这其中就包括我们在应用中声明的各种业务`Bean`，`Bean`的实例化和初始化过程逻辑都在这个函数中。

我想特别介绍下`invokeBeanFactoryPostProcessors(beanFactory)`，我们都知道`Spring`会帮我们自动扫描出所有我们需要放入容器的`Bean`，其具体的实现就在这里。里面会执行所有的`BeanFactoryPostProcessors`，`Spring`帮我们定义了一个`ConfigurationClassPostProcessor`，它的函数`postProcessBeanDefinitionRegistry`会在这个阶段被执行，里面会根据我们配置的`scanning-path`把我们所有使用注解(`@Component、@Service等…`)注释的`Bean`解析成`BeanDefinition`，并记录`Bean`的信息到容器中。

<img src="https://gitee.com/qc_faith/picture/raw/master/image/202212291648250.png" alt="pe_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE1MDM3MjMx,size_16,color_FFFFFF,t_70)" style="zoom: 50%;" />

<img src="https://gitee.com/qc_faith/picture/raw/master/image/202212291649525.png" alt="在这里插入图片描述" style="zoom:50%;" />

<img src="https://gitee.com/qc_faith/picture/raw/master/image/202212291710498.png" alt="在这里插入图片描述" style="zoom:50%;" />

Spring容器的启动阶段就介绍到这里，因为具体的逻辑实在实在太繁杂，而我也就知道个梗概，有想要想要详细了解的同学，可以一个一个方法的去学习了解下。接下来是我们本文的重点内容：bean的实例化和初始化过程。

# 二.Bean的实例化和初始化

大家总是会错误的理解`Bean`的“实例化”和“初始化”过程，总会以为初始化就是对象执行构造函数生成对象实例的过程，其实不然，在初始化阶段实际对象已经实例化出来了，初始化阶段进行的是依赖的注入和执行一些用户自定义的初始化逻辑。对于`Bean`的构建过程，网上有个非常经典的流程图如下：

<img src="https://img-blog.csdnimg.cn/20200823200306184.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE1MDM3MjMx,size_16,color_FFFFFF,t_70#pic_center" alt="在这里插入图片描述" style="zoom:33%;" />

上面的流程图并不是十分准确，但却把整个bean的构建、使用、销毁流程整体勾勒出来了，实际上Bean的构建流程非常复杂，下面会从源码角度讲解
<img src="https://gitee.com/qc_faith/picture/raw/master/image/202212291641424.png" alt="在这里插入图片描述" style="zoom:50%;" />

## Bean的实例化

### finishBeanFactoryInitialization

我们接着上面从finishBeanFactoryInitialization函数看起，因为我们定义的业务Bean实例化和初始化都是在这个函数中实现的。我们进入finishBeanFactoryInitialization函数内部。

<img src="https://gitee.com/qc_faith/picture/raw/master/image/202212291640292.png" alt="在这里插入图片描述" style="zoom: 33%;" />

其中大部分的函数是Bean构建的准备逻辑，实际的构建逻辑在preInstantiateSingletons中，我们继续往下，看preInstantiateSingletons函数。

<img src="https://gitee.com/qc_faith/picture/raw/master/image/202212291640258.png" alt="在这里插入图片描述" style="zoom: 50%;" />

函数会先获取所有需要构建的Bean名称，之前提过在Spring容器启动阶段Bean的相关信息就会被封装成BeanDefinition并注册到容器中。通过bean的RootBeanDefinition判断该bean是否为可构建的类型，很明显可构建的Bean不能是抽象类，不能是接口，也不能是懒加载的bean。之后会判断对象是不是FactoryBean，FactoryBean是一种特殊的Bean，需要特殊处理。

> **FactoryBean简介**
> 在我们平时的开发中经常会使用第三方库相关类，为了避免接口和实现类的耦合，我们通常会使用工厂模式。提供一个工厂类来实例化具体接口实现类。主体对象只需要注入工厂类，具体接口实现类有变更时，我们只需变更工厂类，而主体类无需做任何改动。针对上面的场景，Spring为我们提供了更方便的工具FactoryBean，当某些第三方库不能直接注册到Spring容器时，就可以实现org.springframework.beans.factory.FactoryBean接口，给出自己的对象实例化逻辑。
> 针对FactoryBean而言，其主语是Bean，定语为Factory，它本身也只是一个bean而已，只不过这种类型的bean本身就是用来生产对象的工厂。下面是FactoryBean接口代码，当我们从容器中获取FactoryBean类型的Bean时，容器返回的是FactoryBean所“生产”的对象类型，而非FactoryBean实现本身。
> public interface FactoryBean {
> 	T getObject();	// 返回我们需要生产的对象
> 	Class<?> getObjectType();	// 返回生产对象的类型
> 	default boolean isSingleton();	// 用于标识该对象是否以单例形式存在于容器中
> }

### getBean(beanName)

然后我们进入getBean(beanName)，我们的业务Bean就是通过这个函数构建的。

<img src="https://gitee.com/qc_faith/picture/raw/master/image/202212291629511.png" alt="在这里插入图片描述" style="zoom: 50%;" />

<img src="https://gitee.com/qc_faith/picture/raw/master/image/202212291709685.png" alt="在这里插入图片描述" style="zoom: 50%;" />

经过一次跳转后我们来到了org.springframework.beans.factory.support.AbstractBeanFactory#doGetBean函数，这个函数还是蛮重要的，接下来我们详细介绍下：
在函数内首先会将传递进来的name通过transformedBeanName()函数转换成了beanName，这个函数主要有两个功能：1: 特殊处理FactoryBean的名称、2: 解决别名的问题。
之后我们会看见getSingleton(beanName)函数，这个函数灰常重要，它是用来解决[循环依赖](https://so.csdn.net/so/search?q=循环依赖&spm=1001.2101.3001.7020)的。举个例子：有两个Bean A和B，这两个Bean间存在相互依赖。那么当容器构建A时，实例化好A后，容器会把尚未注入依赖的A暂时放入缓存中，然后去帮A注入依赖。这时容器发现A依赖于B，但是B还没有构建好，那么就会先去构建B。容器在实例化好B时，同样需要帮B注入依赖，此时B发现自己依赖A，在获取A时就可以通过getSingleton(beanName)获取尚未注入依赖的A引用，这样就不会阻塞B的构建，B完成构建后就又可以注入到A中，这样就解决了简单的循环依赖问题。当然循环依赖的问题场景很多，出现的情景也不同，上面举的例子只是最简单的一种，后面会对常见的循环依赖问题进行总结。

<img src="https://gitee.com/qc_faith/picture/raw/master/image/202212291640624.png" alt="在这里插入图片描述" style="zoom: 50%;" />

在之后会通过isPrototypeCurrentlyInCreation(beanName)检测Prototype类型的Bean是否已经正在构建。说到这里，我们首先需要明确的一点是：Prototype类型Bean不支持循环引用 **(包括自我引用)**，而Singleton类型的bean是可以循环引用的 **(包括自我引用)**，这里说的自我引用指的是自己注入自己。其实原因也不难理解，因为Prototype类型的bean是即取即用的，容器不会维护它的生命周期，更不会保存它的引用，那么就不会像Singleton类型的bean一样，在容器中有一份缓存，其实Prototype类型bean也完全没必要有缓存，毕竟容器无需维护它的生命周期。

<img src="https://gitee.com/qc_faith/picture/raw/master/image/202212291629537.png" alt="在这里插入图片描述" style="zoom: 50%;" />

那么代码执行到这里一但发现Prototype类型的bean已经在构建了，便说明出现了循环引用，便直接抛出异常。举个例子如下：

<img src="https://gitee.com/qc_faith/picture/raw/master/image/202212291639155.png" alt="在这里插入图片描述" style="zoom: 50%;" />

<img src="https://gitee.com/qc_faith/picture/raw/master/image/202212291639247.png" alt="在这里插入图片描述" style="zoom: 50%;" />

> 在spring容器中会根据bean定义的scope来创建实例对象，其中我们比较常用的scope定义有两种即：singleton和prototype。
>
> 1. singleton：
>    标记为singleton的bean在容器中只存在一个实例，所有对该对象的引用将共享这个实例。该实例从容器启动构建好后，就一直存活到容器退出，与容器“几乎”拥有相同的“寿命”。
> 2. prototype：
>    容器在接收到该类型对象的请求时，会每次都重新生成一个新的对象实例给请求方。当容器将对象实例返回给请求方后，容器就不再拥有当前返回对象的引用，请求方需要自己负责该对象后续生命周期的管理。有点类似于我们使用new创建对象，但区别在于我们用new创建的对像无法进行依赖注入，需要我们手动注入依赖，而prototype类型bean的实例化和依赖注入容器都会帮我们处理。
>
> **有一点值得注意的是：当一个单例类型的bean引用一个prototype类型的bean时，prototype类型的bean也会变成单例。这主要是因为单例bean的依赖注入只会在容器启动时执行一次。如果要解决这个问题，可以改为手动调用getBean方法，也可以使用@lookup注解。**

再接下来的逻辑其实就比较清晰了，首先获取Bean的定义信息，然后判断下是否有DependsOn依赖的Bean，没错这里就是@DependsOn注解发挥作用的地方，如果我们的Bean在构建前必须要保证某个Bean已经构建好，那么我们就可以使用这个@DependsOn这个注解。我们可以看到实现逻辑其实很简单，就是取出所有依赖的Bean，然后逐个调用getBean接口依次构建。在处理好依赖的Bean后，容器会根据Bean的类型进行构建，Bean主要有Singleton和Prototype两种类型，当然还有很多我们不常用的类型，这里我们就不管了。因为一般我们使用的Bean都是Singleton类型的，所以我们就直接看Singleton类型bean的构建方法。

<img src="https://gitee.com/qc_faith/picture/raw/master/image/202212291639812.png" alt="在这里插入图片描述" style="zoom: 50%;" />

我们可以看到singleton类型bean的构建调用了DefaultSingletonBeanRegistry实现的getSingleton方法。该方法有两个参数，第一个为bean的名字beanName，第二个为ObjectFactory<?> singletonFactory，是一个函数式接口，该函数式接口之后会在getSingleton方法中使用，该函数式接口的主要实现是createBean方法。

<img src="https://gitee.com/qc_faith/picture/raw/master/image/202212291638658.png" alt="在这里插入图片描述" style="zoom: 50%;" />

<img src="https://gitee.com/qc_faith/picture/raw/master/image/202212291629929.png" alt="在这里插入图片描述" style="zoom: 50%;" />

下面是DefaultSingletonBeanRegistry#getSingleton函数的实现代码，其核心的四个函数为：

```java
// 前置检查，通过 构建中bean集合(singletonsCurrentlyInCreation)来检测该bean是否进行了重复构建，
// 可以防止构造器注入可能导致的循环引用问题
beforeSingletonCreation(beanName);
// 调用函数式接口ObjectFactory的具体实现逻辑，即上文中getSingleton方法的第二个参数，主要实现就是createBean方法
singletonFactory.getObject();
// 后置处理，将当前bean从 构建中bean集合(singletonsCurrentlyInCreation)中移除。
afterSingletonCreation(beanName);
// 把结果存在singletonObjects中，并删除一些用于处理循环引用的中间状态
addSingleton(beanName, singletonObject);

public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
		Assert.notNull(beanName, "Bean name must not be null");
		synchronized (this.singletonObjects) {
			// 查看单例bean是否已经构建好，凡是构建好的单例bean，都会存在Map<String, Object> singletonObjects里面
			Object singletonObject = this.singletonObjects.get(beanName);
			if (singletonObject == null) {
				// this.singletonsCurrentlyInDestruction是个boolean类型，记录的是当前这个单例bean是否正在被销毁，
				// 如果是true，代表单例bean已经执行了自身的destroy销毁方法，或者有异常的时候执行了destroySingleton方法等情况
				if (this.singletonsCurrentlyInDestruction) {
					throw new BeanCreationNotAllowedException(beanName,
							"Singleton bean creation not allowed while singletons of this factory are in destruction " +
							"(Do not request a bean from a BeanFactory in a destroy method implementation!)");
				}
				if (logger.isDebugEnabled()) {
					logger.debug("Creating shared instance of singleton bean '" + beanName + "'");
				}
				// 前置检查，通过构建中bean集合(singletonsCurrentlyInCreation)来检测该bean是否进行了重复构建，
				// 可以防止构造器注入可能导致的循环引用问题
				beforeSingletonCreation(beanName);
				boolean newSingleton = false;
				// this.suppressedExceptions是个Set，这个Set里面存放着各种异常信息
				boolean recordSuppressedExceptions = (this.suppressedExceptions == null);
				// 如果此时suppressedExceptions为空，就new LinkedHashSet<>()来保存接下来可能发生的异常
				if (recordSuppressedExceptions) {
					this.suppressedExceptions = new LinkedHashSet<>();
				}
				try {
					// 调用ObjectFactory函数式方法
					singletonObject = singletonFactory.getObject();
					newSingleton = true;
				}
				catch (IllegalStateException ex) {
					// 看这个时候单例是否已经被创建了，如果是的话，不报错，继续执行
					singletonObject = this.singletonObjects.get(beanName);
					// 如果没有被创建，报错
					if (singletonObject == null) {
						throw ex;
					}
				}
				catch (BeanCreationException ex) {
					if (recordSuppressedExceptions) {
						// 把出现过的异常加进去
						for (Exception suppressedException : this.suppressedExceptions) {
							ex.addRelatedCause(suppressedException);
						}
					}
					throw ex;
				}
				finally {
					if (recordSuppressedExceptions) {
						this.suppressedExceptions = null;
					}
					// 后置处理，将当前bean从 构建中bean集合(singletonsCurrentlyInCreation)中移除。
					afterSingletonCreation(beanName);
				}
				if (newSingleton) {
					// 把结果存在singletonObjects中，并删除一些中间状态
					addSingleton(beanName, singletonObject);
				}
			}
			return singletonObject;
		}
	}
```

### createBean

接下来我们进入createBean(beanName, mbd, args)函数中，具体代码如下所示：

<img src="https://gitee.com/qc_faith/picture/raw/master/image/202212291637206.png" alt="在这里插入图片描述" style="zoom: 50%;" />

函数内首先做了些准备工作，包括：根据设置的class属性和className解析class、对override属性进行标记和验证（这里需要注意的是Spring的配置里面根本没有override-method之类的配置，但是在spring配置中存在lookup-method和replace-method，这两个配置会被统一存放在beanDefinition中的methodOverrides属性里，这个方法就是对这两个配置做操作）。在这之后会执行resolveBeforeInstantiation函数，千万不要被它上面的注释误导了，这个函数并不是用来生成我们 **“通常意义”** 上的动态代理的，我们 **“通常意义”** 上的动态代理实现在后面会有介绍。
我们可以看到resolveBeforeInstantiation里面实际上调用了applyBeanPostProcessorsBeforeInstantiation接口，在该接口里面会执行所有InstantiationAwareBeanPostProcessor的postProcessBeforeInstantiation方法。

<img src="https://gitee.com/qc_faith/picture/raw/master/image/202212291636607.png" alt="在这里插入图片描述" style="zoom: 50%;" />

<img src="https://gitee.com/qc_faith/picture/raw/master/image/202212291636793.png" alt="在这里插入图片描述" style="zoom: 50%;" />

在这我们先介绍下InstantiationAwareBeanPostProcessor，大家应该都知道BeanPostProcessor接口，这个接口作用于Bean的构建过程，可以对Bean进行一些自定义的修改，比如属性值的修改、构造器的选择等等。而InstantiationAwareBeanPostProcessor接口继承自BeanPostProcessor接口。新增了3个方法，其类图如下：

<img src="https://gitee.com/qc_faith/picture/raw/master/image/202212291629477.png" alt="在这里插入图片描述" style="zoom:67%;" />

通过名字我们就可以知道，新增的3个接口中前两个分别作用于Bean实例化前和实例化后，最后一个接口postProcessPropertyValues作用于Bean的初始化阶段，主要用来设置对象属性(依赖注入)，其中我们最常用的注入依赖对象的@Autowired注解，就是通过实现这个接口来进行依赖注入的。
我们接着介绍applyBeanPostProcessorsBeforeInstantiation方法，我们可以看到该方法会找到所有InstantiationAwareBeanPostProcessor类型的BeanPostProcessor，然后调用其实现的postProcessBeforeInstantiation的方法。我们最常见的该方法的一个实现类是AbstractAutoProxyCreator，我们看下其具体实现：

<img src="https://gitee.com/qc_faith/picture/raw/master/image/202212291636138.png" alt="在这里插入图片描述" style="zoom: 50%;" />

这里shouldSkip(beanClass, beanName)方法是一个很容易忽略的方法，但这个方法第一次被调用时把所有的切面（Advisor）加载完成。另外我们可以看到，如果我们有自定义的TargetSource，那么在这里就会直接创建代理对象。postProcessBeforeInstantiation()返回对象后，会对该对象执行所有BeanPostProcessor的postProcessorAfterInitialization()方法，最后直接返回对象给用户。

> TargetSource相当于代理中目标对象的容器。当方法调用穿过所有切面逻辑，到达调用链的终点时，本该调用目标对象的方法了。但此时Spring AOP做了点手脚，他并不是直接调用这个目标对象的方法，而是需要通过TargetSource的getTarget()方法获取目标对象，然后再调用目标对象的相应方法。这种方式使得TargetSource拥有了目标对象的控制权，可以控制每次方法调用实际作用到的具体对象实例。

我们继续往下走进入doCreateBean(beanName, mbdToUse, args)方法。在Spring里有个有意思的现象：大部分核心功能的实际实现函数都是以do开头的。现在我们走到了doCreateBean函数，通过名字我们就可以知道该函数用来构建Bean，而实际上**Bean实例化和初始化的核心逻辑都在doCreateBean方法中**，所以接下来的所有讲解都是围绕doCreateBean方法进行的。

```java
/**
	 * Actually create the specified bean. Pre-creation processing has already happened
	 * at this point, e.g. checking {@code postProcessBeforeInstantiation} callbacks.
	 * <p>Differentiates between default bean instantiation, use of a
	 * factory method, and autowiring a constructor.
	 * @param beanName the name of the bean
	 * @param mbd the merged bean definition for the bean
	 * @param args explicit arguments to use for constructor or factory method invocation
	 * @return a new instance of the bean
	 * @throws BeanCreationException if the bean could not be created
	 * @see #instantiateBean
	 * @see #instantiateUsingFactoryMethod
	 * @see #autowireConstructor
	 */
	protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
			throws BeanCreationException {

		// Instantiate the bean.
		BeanWrapper instanceWrapper = null;
		if (mbd.isSingleton()) {
		    // 用于处理factoryBean
			instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
		}
		if (instanceWrapper == null) {
			instanceWrapper = createBeanInstance(beanName, mbd, args);
		}
		final Object bean = instanceWrapper.getWrappedInstance();
		Class<?> beanType = instanceWrapper.getWrappedClass();
		if (beanType != NullBean.class) {
			mbd.resolvedTargetType = beanType;
		}

		// Allow post-processors to modify the merged bean definition.
		synchronized (mbd.postProcessingLock) {
			if (!mbd.postProcessed) {
				try {
					applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
				}
				catch (Throwable ex) {
					throw new BeanCreationException(mbd.getResourceDescription(), beanName,
							"Post-processing of merged bean definition failed", ex);
				}
				mbd.postProcessed = true;
			}
		}

		// Eagerly cache singletons to be able to resolve circular references
		// even when triggered by lifecycle interfaces like BeanFactoryAware.
		boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
				isSingletonCurrentlyInCreation(beanName));
		if (earlySingletonExposure) {
			if (logger.isTraceEnabled()) {
				logger.trace("Eagerly caching bean '" + beanName +
						"' to allow for resolving potential circular references");
			}
			addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
		}

		// Initialize the bean instance.
		Object exposedObject = bean;
		try {
			populateBean(beanName, mbd, instanceWrapper);
			exposedObject = initializeBean(beanName, exposedObject, mbd);
		}
		catch (Throwable ex) {
			if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
				throw (BeanCreationException) ex;
			}
			else {
				throw new BeanCreationException(
						mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
			}
		}

		if (earlySingletonExposure) {
			Object earlySingletonReference = getSingleton(beanName, false);
			if (earlySingletonReference != null) {
				if (exposedObject == bean) {
					exposedObject = earlySingletonReference;
				}
				else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
					String[] dependentBeans = getDependentBeans(beanName);
					Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
					for (String dependentBean : dependentBeans) {
						if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
							actualDependentBeans.add(dependentBean);
						}
					}
					if (!actualDependentBeans.isEmpty()) {
						throw new BeanCurrentlyInCreationException(beanName,
								"Bean with name '" + beanName + "' has been injected into other beans [" +
								StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
								"] in its raw version as part of a circular reference, but has eventually been " +
								"wrapped. This means that said other beans do not use the final version of the " +
								"bean. This is often the result of over-eager type matching - consider using " +
								"'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.");
					}
				}
			}
		}

		// Register bean as disposable.
		try {
			registerDisposableBeanIfNecessary(beanName, bean, mbd);
		}
		catch (BeanDefinitionValidationException ex) {
			throw new BeanCreationException(
					mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
		}

		return exposedObject;
	}
```

这个函数的主体逻辑如下：

- **实例化bean，并封装进BeanWrapper中**
- **执行所有MergedBeanDefinitionPostProcessor处理器**
- **进行依赖注入**
- **执行初始化逻辑，及对bean进行属性修改及切面增强**
- **注册Bean的销毁逻辑**
- **返回构建好的bean**

> BeanWrapper相当于一个代理器，Spring委托BeanWrapper完成Bean属性的填充工作。BeanWrapper继承了PropertyAccessor和PropertyEditorRegistry。BeanWrapperImpl是BeanWrapper接口的默认实现，它会缓存Bean的内省结果而提高效率。BeanWrapper对bean实例的操作很方便，可以免去直接使用java反射API带来的繁琐。

### createBeanInstance

factoryBean的处理逻辑忽略，接下来我们看createBeanInstance方法，该方法主要用来实例化bean，bean实例化的核心逻辑就在该方法中。

```java
/**
	 * Create a new instance for the specified bean, using an appropriate instantiation strategy:
	 * factory method, constructor autowiring, or simple instantiation.
	 * @param beanName the name of the bean
	 * @param mbd the bean definition for the bean
	 * @param args explicit arguments to use for constructor or factory method invocation
	 * @return a BeanWrapper for the new instance
	 * @see #obtainFromSupplier
	 * @see #instantiateUsingFactoryMethod
	 * @see #autowireConstructor
	 * @see #instantiateBean
	 */
	protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) {
		// Make sure bean class is actually resolved at this point.(确保此时实际解析了 Bean 类。)
		Class<?> beanClass = resolveBeanClass(mbd, beanName);
        // 确保class不为空，并且访问权限为public
		if (beanClass != null && !Modifier.isPublic(beanClass.getModifiers()) && !mbd.isNonPublicAccessAllowed()) {
			throw new BeanCreationException(mbd.getResourceDescription(), beanName,
					"Bean class isn't public, and non-public access not allowed: " + beanClass.getName());
		}

		Supplier<?> instanceSupplier = mbd.getInstanceSupplier();
		if (instanceSupplier != null) {
			return obtainFromSupplier(instanceSupplier, beanName);
		}

		if (mbd.getFactoryMethodName() != null) {
			return instantiateUsingFactoryMethod(beanName, mbd, args);
		}

		// Shortcut when re-creating the same bean...
		// 一个类可能有多个构造器，所以Spring得根据参数个数、类型确定需要调用的构造器
		// 在使用构造器创建实例后，Spring会将解析过后确定下来的构造器或工厂方法保存在缓存中，避免再次创建相同bean时再次解析
		boolean resolved = false;
		boolean autowireNecessary = false;
		if (args == null) {
			synchronized (mbd.constructorArgumentLock) {
				if (mbd.resolvedConstructorOrFactoryMethod != null) {
					resolved = true;
					autowireNecessary = mbd.constructorArgumentsResolved;
				}
			}
		}
		if (resolved) {
			if (autowireNecessary) {
				return autowireConstructor(beanName, mbd, null, null);
			}
			else {
				return instantiateBean(beanName, mbd);
			}
		}

		// Candidate constructors for autowiring?
		// 需要根据参数解析、确定构造函数
		Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
		// 解析的构造器不为空 || 注入类型为构造函数自动注入 || bean定义中有构造器参数 || 传入参数不为空
		if (ctors != null || mbd.getResolvedAutowireMode() == AUTOWIRE_CONSTRUCTOR ||
				mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args)) {
			return autowireConstructor(beanName, mbd, ctors, args);
		}

		// Preferred constructors for default construction?
		ctors = mbd.getPreferredConstructors();
		if (ctors != null) {
			return autowireConstructor(beanName, mbd, ctors, null);
		}

		// No special handling: simply use no-arg constructor.
		return instantiateBean(beanName, mbd);
	}
```

上面方法的目的就是选出一个策略来实例化一个对象，那有什么策略呢？ 这就看程序员是怎么配置的了，程序员可以配置工厂方法，指定构造方法，或者是程序员没有做出任何干涉，让Spring按自己的方式去实例化。
上面代码的主要逻辑如下：

- 如果bean定义中存在 InstanceSupplier ，会使用这个回调接口创建对象（应该是3.X以后新加的，3.X的源码中没有）
- 根据配置的factoryMethodName或factory-method创建bean（通过工厂方法创建对象）
- 解析构造函数并进行实例化

通过回调接口或工厂方法创建对象的情况，我们接触的比较少，我们这里主要介绍使用构造器创建bean实例。
因为一个类可能有多个构造函数，所以需要根据配置文件中配置的参数或传入的参数确定最终调用的构造函数，因为判断过程会比较消耗性能，所以Spring会将解析、确定好的构造函数缓存到BeanDefinition中的resolvedConstructorOrFactoryMethod字段中。在下次创建相同bean的时候，会直接从RootBeanDefinition中的属性resolvedConstructorOrFactoryMethod缓存的值获取，避免再次解析。
determineConstructorsFromBeanPostProcessors方法会帮我们选择合适的构造器，我们进该方法中：

<img src="https://gitee.com/qc_faith/picture/raw/master/image/202212291635760.png" alt="在这里插入图片描述" style="zoom: 50%;" />

在这段代码中会使用SmartInstantiationAwareBeanPostProcessor类型的BeanPostProcessor选择合适的构造器，目前该功能主要由SmartInstantiationAwareBeanPostProcessor的子类AutowiredAnnotationBeanPostProcessor实现，下面是相关代码。

```java
@Override
	@Nullable
	public Constructor<?>[] determineCandidateConstructors(Class<?> beanClass, final String beanName)
			throws BeanCreationException {

		// Let's check for lookup methods here...
		if (!this.lookupMethodsChecked.contains(beanName)) {
			if (AnnotationUtils.isCandidateClass(beanClass, Lookup.class)) {
				try {
					Class<?> targetClass = beanClass;
					do {
						ReflectionUtils.doWithLocalMethods(targetClass, method -> {
							Lookup lookup = method.getAnnotation(Lookup.class);
							if (lookup != null) {
								Assert.state(this.beanFactory != null, "No BeanFactory available");
								LookupOverride override = new LookupOverride(method, lookup.value());
								try {
									RootBeanDefinition mbd = (RootBeanDefinition)
											this.beanFactory.getMergedBeanDefinition(beanName);
									mbd.getMethodOverrides().addOverride(override);
								}
								catch (NoSuchBeanDefinitionException ex) {
									throw new BeanCreationException(beanName,
											"Cannot apply @Lookup to beans without corresponding bean definition");
								}
							}
						});
						targetClass = targetClass.getSuperclass();
					}
					while (targetClass != null && targetClass != Object.class);

				}
				catch (IllegalStateException ex) {
					throw new BeanCreationException(beanName, "Lookup method resolution failed", ex);
				}
			}
			this.lookupMethodsChecked.add(beanName);
		}

   //先查找缓存
   Constructor<?>[] candidateConstructors = this.candidateConstructorsCache.get(beanClass);
   //若是缓存中没有
   if (candidateConstructors == null) {
       // Fully synchronized resolution now...
       //同步此方法
       synchronized (this.candidateConstructorsCache) {
           candidateConstructors = this.candidateConstructorsCache.get(beanClass);
           //双重判断，避免多线程并发问题
           if (candidateConstructors == null) {
               Constructor<?>[] rawCandidates;
               try {
                   //获取此Bean的所有构造器
                   rawCandidates = beanClass.getDeclaredConstructors();
               }
               catch (Throwable ex) {
                   throw new BeanCreationException(beanName, "Resolution of declared constructors on bean Class [" +              beanClass.getName() + "] from ClassLoader [" + beanClass.getClassLoader() + "] failed", ex);
               }
               //最终适用的构造器集合
               List<Constructor<?>> candidates = new ArrayList<>(rawCandidates.length);
               //存放依赖注入的required=true的构造器
               Constructor<?> requiredConstructor = null;
               //存放默认构造器
               Constructor<?> defaultConstructor = null;
               Constructor<?> primaryConstructor = BeanUtils.findPrimaryConstructor(beanClass);
               int nonSyntheticConstructors = 0;
               for (Constructor<?> candidate : rawCandidates) {
                   if (!candidate.isSynthetic()) {
                       nonSyntheticConstructors++;
                   }
                   else if (primaryConstructor != null) {
                       continue;
                   }
                   //查找当前构造器上的注解
                   AnnotationAttributes ann = findAutowiredAnnotation(candidate);
                   //若没有注解
                   if (ann == null) {
                       Class<?> userClass = ClassUtils.getUserClass(beanClass);
                       if (userClass != beanClass) {
                           try {
                               Constructor<?> superCtor =
                                   userClass.getDeclaredConstructor(candidate.getParameterTypes());
                               ann = findAutowiredAnnotation(superCtor);
                           }
                           catch (NoSuchMethodException ex) {
                               // Simply proceed, no equivalent superclass constructor found...
                           }
                       }
                   }
                   //若有注解
                   if (ann != null) {
                       //已经存在一个required=true的构造器了，抛出异常
                       if (requiredConstructor != null) {
                           throw new BeanCreationException(beanName, "Invalid autowire-marked constructor: " + candidate + ". Found constructor with 'required' Autowired annotation already: " + requiredConstructor);
                       }
                       //判断此注解上的required属性
                       boolean required = determineRequiredStatus(ann);
                       //若为true
                       if (required) {
                           if (!candidates.isEmpty()) {
                               throw new BeanCreationException(beanName, "Invalid autowire-marked constructors: " + candidates + ". Found constructor with 'required' Autowired annotation: " + candidate);
                           }
                           //将当前构造器加入requiredConstructor集合
                           requiredConstructor = candidate;
                       }
                       //加入适用的构造器集合中
                       candidates.add(candidate);
                   }
                   //如果该构造函数上没有注解，再判断构造函数上的参数个数是否为0
                   else if (candidate.getParameterCount() == 0) {
                       //如果没有参数，加入defaultConstructor集合
                       defaultConstructor = candidate;
                   }
               }
               //适用的构造器集合若不为空
               if (!candidates.isEmpty()) {
                   // Add default constructor to list of optional constructors, as fallback.
                   //若没有required=true的构造器
                   if (requiredConstructor == null) {
                       if (defaultConstructor != null) {
                           //将defaultConstructor集合的构造器加入适用构造器集合
                           candidates.add(defaultConstructor);
                       }
                       else if (candidates.size() == 1 && logger.isInfoEnabled()) {
                           logger.info("Inconsistent constructor declaration on bean with name '" + beanName +
                                       "': single autowire-marked constructor flagged as optional - " +
                                       "this constructor is effectively required since there is no " +
                                       "default constructor to fall back to: " + candidates.get(0));
                       }
                   }
                   //将适用构造器集合赋值给将要返回的构造器集合
                   candidateConstructors = candidates.toArray(new Constructor<?>[0]);
               }
               //如果适用的构造器集合为空，且Bean只有一个构造器并且此构造器参数数量大于0
               else if (rawCandidates.length == 1 && rawCandidates[0].getParameterCount() > 0) {
                   //就使用此构造器来初始化
                   candidateConstructors = new Constructor<?>[] {rawCandidates[0]};
               }
               //如果构造器有两个，且默认构造器不为空
               else if (nonSyntheticConstructors == 2 && primaryConstructor != null &&
                        defaultConstructor != null && !primaryConstructor.equals(defaultConstructor)) {
                   //使用默认构造器返回
                   candidateConstructors = new Constructor<?>[] {primaryConstructor, defaultConstructor};
               }
               else if (nonSyntheticConstructors == 1 && primaryConstructor != null) {
                   candidateConstructors = new Constructor<?>[] {primaryConstructor};
               }
               else {
                   //上述都不符合的话，只能返回一个空集合了
                   candidateConstructors = new Constructor<?>[0];
               }
               //放入缓存，方便下一次调用
               this.candidateConstructorsCache.put(beanClass, candidateConstructors);
           }
       }
   }
   //返回candidateConstructors集合，若为空集合返回null
   return (candidateConstructors.length > 0 ? candidateConstructors : null);
}
```

上面的代码首先会查看这个将被实例化的对象中有没有添加了@Lookup注解的方法，有的话为这种方法生成代理。之后会循环遍历所有的构造方法，以选取合适的构造方法。这里面的逻辑非常复杂，这里就不做介绍了。
这里顺便提下，在Spring中当我们使用lombok的@Builder注解时，**该类默认不带无参构造函数。** 若此时使用fastJson将json字符串转换为实体类，将抛出异常，需要增加@NoArgsConstructor注解防止出现异常。

<img src="https://gitee.com/qc_faith/picture/raw/master/image/202212291635520.png" alt="在这里插入图片描述" style="zoom: 50%;" />

### 对象的实例化

在选定好了构造参数后，我们接下来看对象的实例化过程，Spring的实例化可以分为 **【带参实例化】** 及 **【无参实例化】** 。我们先来看下带参实例化方法autowireConstructor：

```csharp
/**
	 * "autowire constructor" (with constructor arguments by type) behavior.
	 * Also applied if explicit constructor argument values are specified,
	 * matching all remaining arguments with beans from the bean factory.
	 * <p>This corresponds to constructor injection: In this mode, a Spring
	 * bean factory is able to host components that expect constructor-based
	 * dependency resolution.
	 * @param beanName the name of the bean
	 * @param mbd the merged bean definition for the bean
	 * @param chosenCtors chosen candidate constructors (or {@code null} if none)
	 * @param explicitArgs argument values passed in programmatically via the getBean method,
	 * or {@code null} if none (-> use constructor argument values from bean definition)
	 * @return a BeanWrapper for the new instance
	 */
	public BeanWrapper autowireConstructor(String beanName, RootBeanDefinition mbd,
			@Nullable Constructor<?>[] chosenCtors, @Nullable Object[] explicitArgs) {

		BeanWrapperImpl bw = new BeanWrapperImpl();
		this.beanFactory.initBeanWrapper(bw);

		Constructor<?> constructorToUse = null;
		ArgumentsHolder argsHolderToUse = null;
		Object[] argsToUse = null;

		if (explicitArgs != null) {
			argsToUse = explicitArgs;
		}
		else {
			Object[] argsToResolve = null;
			synchronized (mbd.constructorArgumentLock) {
				constructorToUse = (Constructor<?>) mbd.resolvedConstructorOrFactoryMethod;
				if (constructorToUse != null && mbd.constructorArgumentsResolved) {
					// Found a cached constructor...
					argsToUse = mbd.resolvedConstructorArguments;
					if (argsToUse == null) {
						argsToResolve = mbd.preparedConstructorArguments;
					}
				}
			}
			if (argsToResolve != null) {
				argsToUse = resolvePreparedArguments(beanName, mbd, bw, constructorToUse, argsToResolve, true);
			}
		}

		if (constructorToUse == null || argsToUse == null) {
			// Take specified constructors, if any.
			Constructor<?>[] candidates = chosenCtors;
			if (candidates == null) {
				Class<?> beanClass = mbd.getBeanClass();
				try {
					candidates = (mbd.isNonPublicAccessAllowed() ?
							beanClass.getDeclaredConstructors() : beanClass.getConstructors());
				}
				catch (Throwable ex) {
					throw new BeanCreationException(mbd.getResourceDescription(), beanName,
							"Resolution of declared constructors on bean Class [" + beanClass.getName() +
							"] from ClassLoader [" + beanClass.getClassLoader() + "] failed", ex);
				}
			}

			if (candidates.length == 1 && explicitArgs == null && !mbd.hasConstructorArgumentValues()) {
				Constructor<?> uniqueCandidate = candidates[0];
				if (uniqueCandidate.getParameterCount() == 0) {
					synchronized (mbd.constructorArgumentLock) {
						mbd.resolvedConstructorOrFactoryMethod = uniqueCandidate;
						mbd.constructorArgumentsResolved = true;
						mbd.resolvedConstructorArguments = EMPTY_ARGS;
					}
					bw.setBeanInstance(instantiate(beanName, mbd, uniqueCandidate, EMPTY_ARGS));
					return bw;
				}
			}

			// Need to resolve the constructor.
			boolean autowiring = (chosenCtors != null ||
					mbd.getResolvedAutowireMode() == AutowireCapableBeanFactory.AUTOWIRE_CONSTRUCTOR);
			ConstructorArgumentValues resolvedValues = null;

			int minNrOfArgs;
			if (explicitArgs != null) {
				minNrOfArgs = explicitArgs.length;
			}
			else {
				ConstructorArgumentValues cargs = mbd.getConstructorArgumentValues();
				resolvedValues = new ConstructorArgumentValues();
				minNrOfArgs = resolveConstructorArguments(beanName, mbd, bw, cargs, resolvedValues);
			}

			AutowireUtils.sortConstructors(candidates);
			int minTypeDiffWeight = Integer.MAX_VALUE;
			Set<Constructor<?>> ambiguousConstructors = null;
			LinkedList<UnsatisfiedDependencyException> causes = null;

			for (Constructor<?> candidate : candidates) {
				Class<?>[] paramTypes = candidate.getParameterTypes();

				if (constructorToUse != null && argsToUse != null && argsToUse.length > paramTypes.length) {
					// Already found greedy constructor that can be satisfied ->
					// do not look any further, there are only less greedy constructors left.
					break;
				}
				if (paramTypes.length < minNrOfArgs) {
					continue;
				}

				ArgumentsHolder argsHolder;
				if (resolvedValues != null) {
					try {
						String[] paramNames = ConstructorPropertiesChecker.evaluate(candidate, paramTypes.length);
						if (paramNames == null) {
							ParameterNameDiscoverer pnd = this.beanFactory.getParameterNameDiscoverer();
							if (pnd != null) {
								paramNames = pnd.getParameterNames(candidate);
							}
						}
						argsHolder = createArgumentArray(beanName, mbd, resolvedValues, bw, paramTypes, paramNames,
								getUserDeclaredConstructor(candidate), autowiring, candidates.length == 1);
					}
					catch (UnsatisfiedDependencyException ex) {
						if (logger.isTraceEnabled()) {
							logger.trace("Ignoring constructor [" + candidate + "] of bean '" + beanName + "': " + ex);
						}
						// Swallow and try next constructor.
						if (causes == null) {
							causes = new LinkedList<>();
						}
						causes.add(ex);
						continue;
					}
				}
				else {
					// Explicit arguments given -> arguments length must match exactly.
					if (paramTypes.length != explicitArgs.length) {
						continue;
					}
					argsHolder = new ArgumentsHolder(explicitArgs);
				}

				int typeDiffWeight = (mbd.isLenientConstructorResolution() ?
						argsHolder.getTypeDifferenceWeight(paramTypes) : argsHolder.getAssignabilityWeight(paramTypes));
				// Choose this constructor if it represents the closest match.
				if (typeDiffWeight < minTypeDiffWeight) {
					constructorToUse = candidate;
					argsHolderToUse = argsHolder;
					argsToUse = argsHolder.arguments;
					minTypeDiffWeight = typeDiffWeight;
					ambiguousConstructors = null;
				}
				else if (constructorToUse != null && typeDiffWeight == minTypeDiffWeight) {
					if (ambiguousConstructors == null) {
						ambiguousConstructors = new LinkedHashSet<>();
						ambiguousConstructors.add(constructorToUse);
					}
					ambiguousConstructors.add(candidate);
				}
			}

			if (constructorToUse == null) {
				if (causes != null) {
					UnsatisfiedDependencyException ex = causes.removeLast();
					for (Exception cause : causes) {
						this.beanFactory.onSuppressedException(cause);
					}
					throw ex;
				}
				throw new BeanCreationException(mbd.getResourceDescription(), beanName,
						"Could not resolve matching constructor " +
						"(hint: specify index/type/name arguments for simple parameters to avoid type ambiguities)");
			}
			else if (ambiguousConstructors != null && !mbd.isLenientConstructorResolution()) {
				throw new BeanCreationException(mbd.getResourceDescription(), beanName,
						"Ambiguous constructor matches found in bean '" + beanName + "' " +
						"(hint: specify index/type/name arguments for simple parameters to avoid type ambiguities): " +
						ambiguousConstructors);
			}

			if (explicitArgs == null && argsHolderToUse != null) {
				argsHolderToUse.storeCache(mbd, constructorToUse);
			}
		}

		Assert.state(argsToUse != null, "Unresolved constructor arguments");
		bw.setBeanInstance(instantiate(beanName, mbd, constructorToUse, argsToUse));
		return bw;
	}
```

上面代码非常长，总体的功能逻辑如下：

1. **确定实例化时构造器传入的参数：**
   （1）如果调用getBean方式时传入的参数不为空，则可以直接使用传入的参数。
   （2）否则尝试从缓存中获取参数。
   （3）否则解析xml中配置的构造器参数。

2. **确定使用的构造函数:**
   根据第一步中确定下来的参数，接下来的任务就是根据参数的个数、类型来确定最终调用的构造函数。首先是根据参数个数匹配，把所有构造函数根据参数个数升序排序，筛选出参数个数匹配的构造函数。因为配置文件中可以通过参数位置索引，也可以通过参数名称来设定参数值，所以还需要解析参数的名称。最后根据解析好的参数名称、参数类型、实际参数就可以确定构造函数，并且将参数转换成对应的类型。

3. **构造函数不确定性的验证。** 因为有一些构造函数的参数类型为父子关系，所以Spring会做一次验证。

4. **如果条件符合（传入参数为空），将解析好的构造函数、参数放入缓存。**

5. **根据实例化策略通过构造函数、参数实例化bean。**
   接下来我们看下对象的实例化策略。

   <img src="https://gitee.com/qc_faith/picture/raw/master/image/202212291629926.png" alt="在这里插入图片描述" style="zoom: 50%;" />

   对象的实例化主要有两种策略：SimpleInstantiationStrategy和CglibSubclassingInstantiationStrategy。其中SimpleInstantiationStrategy通过反射方式创建对象，而CglibSubclassingInstantiationStrategy通过Cglib来创建对象。接下来看下无参实例化的方法：

   <img src="https://gitee.com/qc_faith/picture/raw/master/image/202212291634207.png" alt="在这里插入图片描述" style="zoom:50%;" />

   可以看到无参实例化的方法和带有参数创建bean一样，都是使用InstantiationStrategy来创建bean，但是因为没有参数，所以也就没有必要执行繁琐的确定构造函数的代码，只需要使用无参构造器进行实例化就行了。

   <img src="https://gitee.com/qc_faith/picture/raw/master/image/202212291630780.png" alt="在这里插入图片描述" style="zoom:50%;" />

   到这里我们就把createBeanInstance的逻辑讲完了，同时整个Bean的实例化流程也结束了。到此时，bean已经被实例化了出来，但属性依赖还没有注入，是一个纯净的原生对象。接下来spring会帮我们对bean进行依赖注入及功能增强等等等，那我们就一起进入bean的初始化流程吧！

## Bean的初始化

### applyMergedBeanDefinitionPostProcessors

接着上面的逻辑我们继续往下看，来看applyMergedBeanDefinitionPostProcessors方法。

<img src="https://gitee.com/qc_faith/picture/raw/master/image/202212291629689.png" alt="在这里插入图片描述" style="zoom:50%;" />

<img src="https://img-blog.csdnimg.cn/2020080816421739.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE1MDM3MjMx,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:50%;" />

我们可以看到这个函数调用了所有MergedBeanDefinitionPostProcessor的MergedBeanDefinitionPostProcessor方法。MergedBeanDefinitionPostProcessor有很多实现类，其中AutowiredAnnotationBeanPostProcessor是一个非常重要的实现类。主要用来解析所有扫描类里面的@Autowired和@Value注解。其实现的主要方法如下：

<img src="https://gitee.com/qc_faith/picture/raw/master/image/202212291629168.png" alt="在这里插入图片描述" style="zoom:50%;" />

AutowiredAnnotationBeanPostProcessor在构造时就已经把@Autowired和@Value注解类加入到了autowiredAnnotationTypes集合中。 AutowiredAnnotationBeanPostProcessor会对bean进行注解扫描，如果扫描到了@Autowired和@Value注解，就会把对应的方法或者属性封装起来，最终封装成InjectionMetadata对象。

<img src="https://gitee.com/qc_faith/picture/raw/master/image/202212291706204.png" alt="在这里插入图片描述" style="zoom:50%;" />

<img src="https://gitee.com/qc_faith/picture/raw/master/image/202212291634516.png" alt="在这里插入图片描述" style="zoom:50%;" />



### addSingletonFactory

看过了applyMergedBeanDefinitionPostProcessors后，我们继续往下看addSingletonFactory方法，这个方法也是用来解决循环依赖问题的，这里的addSingletonFactory会将生产该对象的工厂类singletonFactory放入到名为singletonFactories的HashMap中，而工厂类singletonFactory的函数式方法实现，主要就是调用getEarlyBeanReference(beanName, mbd, bean)。上文提过当对象A和对象B存在循环依赖时，若对象A先开始构建，在进行依赖注入时发现依赖对象B，就去构建对象B，对象B在构建时发现需要注入对象A，此时就会从singletonFactories中获取对象A的工厂singletonFactory实例，然后通过工厂实例获取对象A的引用来完成注入。**获取的对象A引用最后会被保存到earlySingletonObjects(HashMap)中**。

<img src="https://gitee.com/qc_faith/picture/raw/master/image/202212291633625.png" alt="在这里插入图片描述" style="zoom: 50%;" />

<img src="https://gitee.com/qc_faith/picture/raw/master/image/202212291706249.png" alt="在这里插入图片描述" style="zoom:50%;" />

<img src="https://gitee.com/qc_faith/picture/raw/master/image/202212291629804.png" alt="在这里插入图片描述" style="zoom:50%;" />

### populateBean

我们接着看populateBean(beanName, mbd, instanceWrapper)方法，这个方法主要进行bean属性的装配工作，即依赖对象的注入。下面的逻辑看着很多，但其实很多逻辑都是为了处理旧版本XML属性配置的，但当我们使用@Autowired时，很多逻辑并不会执行。例如下面的pvs是一个MutablePropertyValues实例，里面实现了PropertyValues接口，提供属性的读写操作实现，同时可以通过调用构造函数实现深拷贝，但这个pvs主要是用来处理XML配置的属性的，当我们使用@Autowired时，其值基本都是null或者为空。（深拷贝：从源对象完美复制出一个相同却与源对象彼此独立的目标对象，这里的相同是指两个对象的状态和动作相同，彼此独立是指改变其中一个对象的状态不会影响到另外一个对象。）

```java
    /**
	 * Populate the bean instance in the given BeanWrapper with the property values
	 * from the bean definition.
	 * @param beanName the name of the bean
	 * @param mbd the bean definition for the bean
	 * @param bw the BeanWrapper with bean instance
	 */
	@SuppressWarnings("deprecation")  // for postProcessPropertyValues
	protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
		if (bw == null) {
			if (mbd.hasPropertyValues()) {
				throw new BeanCreationException(
						mbd.getResourceDescription(), beanName, "Cannot apply property values to null instance");
			}
			else {
				// Skip property population phase for null instance.
				return;
			}
		}

		// Give any InstantiationAwareBeanPostProcessors the opportunity to modify the
		// state of the bean before properties are set. This can be used, for example,
		// to support styles of field injection.
		boolean continueWithPropertyPopulation = true;
        
        // 在进行依赖注入前，给InstantiationAwareBeanPostProcessors最后的机会根据用户需求自定义修改Bean的属性值
        // 如果此处返回了false，则后面不会再进行依赖注入。
		if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
			for (BeanPostProcessor bp : getBeanPostProcessors()) {
				if (bp instanceof InstantiationAwareBeanPostProcessor) {
					InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
					if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
						continueWithPropertyPopulation = false;
						break;
					}
				}
			}
		}

		if (!continueWithPropertyPopulation) {
			return;
		}

        // pvs是一个MutablePropertyValues实例，实现了PropertyValues接口，我们配置的对象属性值会被解析到PropertyValues中，
        // 然后通过BeanWrapper进行类型转换并设置给对象，不过PropertyValues主要是用来处理xml配置的属性值的，目前我们基本
        // 使用的都是@Autowired，所以这里的PropertyValues基本都是null。
    	PropertyValues pvs = (mbd.hasPropertyValues() ? mbd.getPropertyValues() : null);

        // 这里的AUTOWIRE_BY_NAME、AUTOWIRE_BY_TYPE主要是针对XML配置的依赖bean自动绑定配置，即<bean>的autowire属性。
        // 指的并不是我们使用的@Autowired，@Autowire并不会走下面这段逻辑，所以我们可以暂时不用看下面逻辑。
		int resolvedAutowireMode = mbd.getResolvedAutowireMode();
		if (resolvedAutowireMode == AUTOWIRE_BY_NAME || resolvedAutowireMode == AUTOWIRE_BY_TYPE) {
			MutablePropertyValues newPvs = new MutablePropertyValues(pvs);
			// Add property values based on autowire by name if applicable.
			if (resolvedAutowireMode == AUTOWIRE_BY_NAME) {
				autowireByName(beanName, mbd, bw, newPvs);
			}
			// Add property values based on autowire by type if applicable.
			if (resolvedAutowireMode == AUTOWIRE_BY_TYPE) {
				autowireByType(beanName, mbd, bw, newPvs);
			}
			pvs = newPvs;
		}

		boolean hasInstAwareBpps = hasInstantiationAwareBeanPostProcessors();
		// 是否进行依赖检查，默认值就是None，所以此处返回false，表示不需要依赖检查
		boolean needsDepCheck = (mbd.getDependencyCheck() != AbstractBeanDefinition.DEPENDENCY_CHECK_NONE);

		PropertyDescriptor[] filteredPds = null;
		if (hasInstAwareBpps) {
			if (pvs == null) {
				pvs = mbd.getPropertyValues();
			}
			for (BeanPostProcessor bp : getBeanPostProcessors()) {
				if (bp instanceof InstantiationAwareBeanPostProcessor) {
					InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
					// 调用InstantiationAwareBeanPostProcessor#postProcessPropertyValues方法，该方法有两个重要的实现：
					// AutowiredAnnotationBeanPostProcessor和CommonAnnotationBeanPostProcessor，前者会通过该方法注入
				  // 所有@Autowired注解的对象，而后者通过该方法注入所有@Resource注解的对象。
					PropertyValues pvsToUse = ibp.postProcessProperties(pvs, bw.getWrappedInstance(), beanName);
					if (pvsToUse == null) {
						if (filteredPds == null) {
							filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
						}
						pvsToUse = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
						if (pvsToUse == null) {
							return;
						}
					}
					pvs = pvsToUse;
				}
			}
		}
		if (needsDepCheck) {
			if (filteredPds == null) {
				filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
			}
			checkDependencies(beanName, mbd, filteredPds, pvs);
		}

        // 将pvs上所有的属性填充到BeanWrapper对应的Bean实例中，前面说过了这里主要是用来填充XML配置的属性，
        // 所以若我们使用@Autowired时pvs为null，这里并不会执行。
		if (pvs != null) {
			applyPropertyValues(beanName, mbd, bw, pvs);
		}
	}
```

我们接下来看postProcessPropertyValues方法，@Autowired和@Resource注解对象的注入就是通过该方法实现的，上面其实已经提过了他们的实现类分别为AutowiredAnnotationBeanPostProcessor和CommonAnnotationBeanPostProcessor。两者的实现逻辑基本相同，我们这里仅介绍@Autowired的实现。

<img src="https://gitee.com/qc_faith/picture/raw/master/image/202212291633988.png" alt="在这里插入图片描述" style="zoom:50%;" />

对象中@Autowired 注入点（字段或方法）都会被提前解析成元信息并保存到InjectionMetadata中。我们可以看到这里的逻辑：依次遍历所有的注入点元信息 InjectedElement，进行属性注入。（下图中的checkedElements主要是为了防止重复注入）

<img src="https://gitee.com/qc_faith/picture/raw/master/image/202212291633511.png" alt="在这里插入图片描述" style="zoom:50%;" />

我们继续跟进element.inject(target, beanName, pvs)方法，这个方法有两个实现：AutowiredFieldElement和AutowiredMethodElement，他们分别用于处理属性注入和方法注入，我们这里只看AutowiredFieldElement，两者实现逻辑基本相同。

<img src="https://gitee.com/qc_faith/picture/raw/master/image/202212291633309.png" alt="在这里插入图片描述" style="zoom:50%;" />

我们可以看到主体逻辑其实很简单：首先解析依赖的属性对象，然后通过反射设置属性。其中beanFactory.resolveDependency()是Spring 进行依赖查找的核心 API。

<img src="https://gitee.com/qc_faith/picture/raw/master/image/202212291629910.png" alt="在这里插入图片描述" style="zoom:50%;" />

beanFactory.resolveDependency()的实际功能由doResolveDependency()函数实现，其主要功能为：

- 快速查找： @Autowired 注解处理场景。AutowiredAnnotationBeanPostProcessor 处理 @Autowired 注解时，如果注入的对象只有一个，会将该 bean 对应的名称缓存起来，下次直接通过名称查找会快很多。
- 注入指定值：@Value 注解处理场景。QualifierAnnotationAutowireCandidateResolver 处理 @Value 注解时，会读取 @Value 对应的值进行注入。如果是 String 要经过三个过程：①占位符处理 -> ②EL 表达式解析 -> ③类型转换，这也是一般的处理过程，BeanDefinitionValueResolver 处理 String 对象也是这个过程。
- 集合依赖查询：直接全部委托给 resolveMultipleBeans 方法。
- 单个依赖查询：先调用 findAutowireCandidates 查找所有可用的依赖，如果有多个依赖，则根据规则匹配： @Primary -> @Priority -> ③方法名称或字段名称。

```java
public Object doResolveDependency(DependencyDescriptor descriptor, String beanName, Set<String> autowiredBeanNames, TypeConverter typeConverter) throws BeansException {

    InjectionPoint previousInjectionPoint = ConstructorResolver.setCurrentInjectionPoint(descriptor);
    try {
        // 1. 快速查找，根据名称查找。AutowiredAnnotationBeanPostProcessor用到
        Object shortcut = descriptor.resolveShortcut(this);
        if (shortcut != null) {
            return shortcut;
        }

        // 2. 注入指定值，QualifierAnnotationAutowireCandidateResolver解析@Value会用到
        Class<?> type = descriptor.getDependencyType();
        Object value = getAutowireCandidateResolver().getSuggestedValue(descriptor);
        if (value != null) {
            if (value instanceof String) {
                // 2.1 占位符解析
                String strVal = resolveEmbeddedValue((String) value);
                BeanDefinition bd = (beanName != null && containsBean(beanName) ?
                                     getMergedBeanDefinition(beanName) : null);
                // 2.2 Spring EL 表达式
                value = evaluateBeanDefinitionString(strVal, bd);
            }
            TypeConverter converter = (typeConverter != null ? typeConverter : getTypeConverter());
            try {
                // 2.3 类型转换
                return converter.convertIfNecessary(value, type, descriptor.getTypeDescriptor());
            } catch (UnsupportedOperationException ex) {
                return (descriptor.getField() != null ?
                        converter.convertIfNecessary(value, type, descriptor.getField()) :
                        converter.convertIfNecessary(value, type, descriptor.getMethodParameter()));
            }
        }

        // 3. 集合依赖，如 Array、List、Set、Map。内部查找依赖也是使用findAutowireCandidates
        Object multipleBeans = resolveMultipleBeans(descriptor, beanName, autowiredBeanNames, typeConverter);
        if (multipleBeans != null) {
            return multipleBeans;
        }

        // 4. 单个依赖查询
        Map<String, Object> matchingBeans = findAutowireCandidates(beanName, type, descriptor);
        // 4.1 没有查找到依赖，判断descriptor.require
        if (matchingBeans.isEmpty()) {
            if (isRequired(descriptor)) {
                raiseNoMatchingBeanFound(type, descriptor.getResolvableType(), descriptor);
            }
            return null;
        }

        String autowiredBeanName;
        Object instanceCandidate;

        // 4.2 有多个，如何过滤
        if (matchingBeans.size() > 1) {
            // 4.2.1 @Primary -> @Priority -> 方法名称或字段名称匹配 
            autowiredBeanName = determineAutowireCandidate(matchingBeans, descriptor);
            // 4.2.2 根据是否必须，抛出异常。注意这里如果是集合处理，则返回null
            if (autowiredBeanName == null) {
                if (isRequired(descriptor) || !indicatesMultipleBeans(type)) {
                    return descriptor.resolveNotUnique(descriptor.getResolvableType(), matchingBeans);
                } else {
                    return null;
                }
            }
            instanceCandidate = matchingBeans.get(autowiredBeanName);
        } else {
            // We have exactly one match.
            Map.Entry<String, Object> entry = matchingBeans.entrySet().iterator().next();
            autowiredBeanName = entry.getKey();
            instanceCandidate = entry.getValue();
        }

        // 4.3 到了这，说明有且仅有命中一个
        if (autowiredBeanNames != null) {
            autowiredBeanNames.add(autowiredBeanName);
        }
        // 4.4 实际上调用 getBean(autowiredBeanName, type)。但什么情况下会出现这种场景？
        if (instanceCandidate instanceof Class) {
            instanceCandidate = descriptor.resolveCandidate(autowiredBeanName, type, this);
        }
        Object result = instanceCandidate;
        if (result instanceof NullBean) {
            if (isRequired(descriptor)) {
                raiseNoMatchingBeanFound(type, descriptor.getResolvableType(), descriptor);
            }
            result = null;
        }
        if (!ClassUtils.isAssignableValue(type, result)) {
            throw new BeanNotOfRequiredTypeException(autowiredBeanName, type, instanceCandidate.getClass());
        }
        return result;
    } finally {
        ConstructorResolver.setCurrentInjectionPoint(previousInjectionPoint);
    }
}
```

上面代码比较多，其中对我们来说比较重要的是findAutowireCandidates函数和determineAutowireCandidate函数。findAutowireCandidates函数主要用于根据类型查找依赖，大致可以分为三步：先查找内部依赖，再根据类型查找，最后没有可注入的依赖则进行补偿。其中@Qualifier注解就是在这个函数中处理的。

<img src="https://gitee.com/qc_faith/picture/raw/master/image/202212291629918.png" alt="在这里插入图片描述" style="zoom: 50%;" />

另外一个对我们比较重要的函数是determineAutowireCandidate()函数，其实这个函数就是当Spring发现有多个对象满足@Autowired要求的对象类型时，选定注入对象的规则：先按 @Primary 查找，再按 @Priority，最后按方法名称或字段名称查找，直到只有一个 bean 为止。所以我们可以看出：在使用@Autowired注入对象时，会优先找到满足要求类型的所有对象，若有多个对象，则会根据@Primary或@Priority过滤，若仍有多个对象，则最后根据对象名称匹配。

<img src="https://gitee.com/qc_faith/picture/raw/master/image/202212291632947.png" alt="在这里插入图片描述" style="zoom: 50%;" />

当选定好依赖对象后，我们继续往下走会进入descriptor.resolveCandidate方法。

<img src="https://gitee.com/qc_faith/picture/raw/master/image/202212291632084.png" alt="在这里插入图片描述" style="zoom:50%;" />

<img src="https://img-blog.csdnimg.cn/20200814011755921.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE1MDM3MjMx,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:50%;" />

我们可以看到在descriptor.resolveCandidate方法中，其实还是调用了beanFactory.getBean(beanName)方法去问容器要该依赖对象，接下来若该对象还没有构建，则又会开始构建该对象，直到依赖的所有对象都已构建好，然后进行属性注入，并这其实就是Spring完成属性注入的方式。

### initializeBean

在对象实例化完成后且相关属性和依赖设置完成后，接下来便是初始化过程了，初始化过程主要由 initializeBean函数实现，其主要逻辑如下:

- **检查Aware相关接口并设置相关依赖**
- **执行BeanPostProcessor的postProcessBeforeInitialization方法(此处会执行@PostConstruct注解方法，且部分Aware接口会在此处处理)**
- **执行InitializingBean接口的afterPropertiesSet方法及init-method方法**
- **执行BeanPostProcessor的postProcessBeforeInitialization方法**

```java
/**
	 * Initialize the given bean instance, applying factory callbacks
	 * as well as init methods and bean post processors.
	 * <p>Called from {@link #createBean} for traditionally defined beans,
	 * and from {@link #initializeBean} for existing bean instances.
	 * @param beanName the bean name in the factory (for debugging purposes)
	 * @param bean the new bean instance we may need to initialize
	 * @param mbd the bean definition that the bean was created with
	 * (can also be {@code null}, if given an existing bean instance)
	 * @return the initialized bean instance (potentially wrapped)
	 * @see BeanNameAware
	 * @see BeanClassLoaderAware
	 * @see BeanFactoryAware
	 * @see #applyBeanPostProcessorsBeforeInitialization
	 * @see #invokeInitMethods
	 * @see #applyBeanPostProcessorsAfterInitialization
	 */
	protected Object initializeBean(final String beanName, final Object bean, @Nullable RootBeanDefinition mbd) {
		// 检查Aware相关接口并设置相关依赖
		if (System.getSecurityManager() != null) {
			AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
				invokeAwareMethods(beanName, bean);
				return null;
			}, getAccessControlContext());
		}
		else {
			invokeAwareMethods(beanName, bean);
		}

        // BeanPostProcessor前置处理(此处会执行@PostConstruct注解方法，且部分Aware接口会在此处处理）
		Object wrappedBean = bean;
		if (mbd == null || !mbd.isSynthetic()) {
			wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
		}

        // 执行InitializingBean接口的afterPropertiesSet方法及init-method方法
		try {
			invokeInitMethods(beanName, wrappedBean, mbd);
		}
		catch (Throwable ex) {
			throw new BeanCreationException(
					(mbd != null ? mbd.getResourceDescription() : null),
					beanName, "Invocation of init method failed", ex);
		}
		
        // BeanPostProcessor后置处理
		if (mbd == null || !mbd.isSynthetic()) {
			wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
		}

		return wrappedBean;
	}
```

在初始化过程中，Spring首先会检查当前对象是否实现了一系列以Aware命名结尾的接口，如果是的话就将这些Aware接口定义中规定的依赖注入给当前对象实例。下面是invokeAwareMethods方法的源码，涉及的BeanNameAware主要有三个：

- BeanNameAware：注入当前 bean 对应 beanName

- BeanClassLoaderAware：注入加载当前 bean的ClassLoader

- BeanFactoryAware：注入当前BeanFactory容器的引用
  
  
  
  <img src="https://gitee.com/qc_faith/picture/raw/master/image/202212291629615.png" alt="在这里插入图片描述" style="zoom:50%;" />
  
  上面的三个Aware接口只是针对BeanFactory类型的容器而言，对于ApplicationContext类型的容器也存在几个Aware相关接口，不过在检测这些接口并设置相关依赖的实现机理上，与以上几个接口处理方式有所不同，是在下面的applyBeanPostProcessorsBeforeInitialization方法中执行的，具体的实现类是ApplicationContextAwareProcessor。
  我们接下来看BeanPostProcessor的postProcessBeforeInitialization方法，它的实现类有很多，除了ApplicationContextAwareProcessor外我们比较常用的还有InitDestroyAnnotationBeanPostProcessor，CommonAnnotationBeanPostProcessor便是继承了该类用于处理@PostConstruct，在postProcessBeforeInitialization方法中@PostConstruct注解的方法会被执行，有兴趣的同学可以研究下，这里就不详细介绍了。
  
  <img src="https://gitee.com/qc_faith/picture/raw/master/image/202212291705752.png" alt="在这里插入图片描述" style="zoom:50%;" />
  
  再接下来会执行InitializingBean接口的afterPropertiesSet方法及init-method方法，由invokeInitMethods方法实现，具体代码如下图。这里会检查当前对象是否实现了InitializingBean接口，如果是，则会调用其afterPropertiesSet()方法进一步调整对象实例的状态。另外spring还提供了另一种方式来指定自定义的对象初始化操作，即init-method方法，这里也会处理。
  
  <img src="https://gitee.com/qc_faith/picture/raw/master/image/202212291629185.png" alt="在这里插入图片描述" style="zoom:50%;" />
  
  我们继续往下看代码wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName)，生成代理对象的逻辑就在这里面。里面会执行BeanPostProcessor的postProcessAfterInitialization方法。
  
  
  
  <img src="https://gitee.com/qc_faith/picture/raw/master/image/202212291705273.png" alt="在这里插入图片描述" style="zoom:50%;" />
  
  postProcessAfterInitialization方法有个重要的实现类AnnotationAwareAspectJAutoProxyCreator，该类便是动态代理的主要实现类，我们通常意义上的动态代理就是通过它实现的。下面的代码便是AnnotationAwareAspectJAutoProxyCreator实现的postProcessAfterInitialization方法。
  
  <img src="https://gitee.com/qc_faith/picture/raw/master/image/202212291650543.png" alt="在这里插入图片描述" style="zoom:50%;" />
  
  
  
  <img src="https://gitee.com/qc_faith/picture/raw/master/image/202212291629720.png" alt="在这里插入图片描述" style="zoom:50%;" />
  
  wrapIfNecessary中有2个和核心方法：
  **1.getAdvicesAndAdvisorsForBean获取当前bean匹配的增强器**
  获取所有增强器，也就是所有@Aspect注解的Bean
  获取匹配的增强器，也就是根据@Before，@After等注解上的表达式，与当前bean进行匹配
  对匹配的增强器进行扩展和排序，就是按照 @Order 或者 PriorityOrdered 的 getOrder 的数据值进行排序，越小的越靠前
  **2.createProxy为当前bean创建代理**
  Spring aop框架使用了AopProxy对不同代理机制进行适度的抽象，并提供了相应的AopProxy子类实现。目前Spring aop框架提供了JDK动态代理和Cglib动态代理两种AopProxy实现，实现类分别为JdkDynamicAopProxy和ObjenesisCglibAopProxy。不同AopProxy实现的实例化过程采用工厂模式进行封装，即AopProxyFactory，AopProxyFactory会根据传入的AdvisedSupport实例信息来决定生成什么类型的AopProxy实例，目前AopProxyFactory只有一个具体实现类即下面的DefaultAopProxyFactory。

```java
public class DefaultAopProxyFactory implements AopProxyFactory, Serializable {

	@Override
	public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
	    // optimize：主要用于告知代理对象是否需要采取进一步的优化措施，默认值为false，当值为true时会使用cglib进行动态代理。
	    // proxyTargetClass：是否使用基于类的代理即cglib动态代理，默认值为false。
	    // hasNoUserSuppliedProxyInterfaces：是否存在用户自定义代理接口，如果目标实现了接口，可以使用JDK代理（默认），如
	    // 果目标对象没有实现接口，则必须使用CGLIB库。
		if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
			Class<?> targetClass = config.getTargetClass();
			if (targetClass == null) {
				throw new AopConfigException("TargetSource cannot determine target class: " +
						"Either an interface or a target is required for proxy creation.");
			}
			if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
				return new JdkDynamicAopProxy(config);
			}
			return new ObjenesisCglibAopProxy(config);
		}
		else {
			return new JdkDynamicAopProxy(config);
		}
	}

	/**
	 * Determine whether the supplied {@link AdvisedSupport} has only the
	 * {@link org.springframework.aop.SpringProxy} interface specified
	 * (or no proxy interfaces specified at all).
	 */
	private boolean hasNoUserSuppliedProxyInterfaces(AdvisedSupport config) {
		Class<?>[] ifcs = config.getProxiedInterfaces();
		return (ifcs.length == 0 || (ifcs.length == 1 && SpringProxy.class.isAssignableFrom(ifcs[0])));
	}
}
```

jdk动态代理和cglib创建动态代理的区别如下：

| 类型   | jdk动态代理                                                  | cglib动态代理                                                |
| ------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 原理   | java动态代理是利用反射机制生成一个实现代理接口的匿名类，在调用具体方法前调用InvokeHandler来处理 | cglib动态代理是利用asm开源包，将代理对象类的class文件加载进来，通过修改其字节码生成子类来处理 |
| 核心类 | Proxy创建代理利用反射机制生成一个实现代理接口的匿名类InvocationHandler方法拦截器接口，需要实现invoke方法 | net.sf.cglib.proxy.Enhancer：主要增强类，通过字节码技术动态创建委托类的子类实例net.sf.cglib.proxy.MethodInterceptor：方法拦截器接口，需要实现intercept方法 |
| 局限性 | 只能代理实现了接口的类                                       | 不能对final修饰的类进行代理，也不能处理final或private修饰的方法，但是可以处理static方法 |

<img src="https://gitee.com/qc_faith/picture/raw/master/image/202212291650675.png" alt="在这里插入图片描述" style="zoom:50%;" />

在创建好AopProxy实例后，便可以通过AopProxy实例创建动态代理，下面是CglibAopProxy#getProxy方法。主要是通过继承的方式生成代理对象的子类，通过覆写父类方法来完成代理。cglib不需要像jdk动态代理那样要求代理类必须实现某个接口，同时性能也更好，但是使用cglib时一定要注意：**cglib无法对final或private修饰的方法进行代理，但可以对static修饰的方法进行代理**。

<img src="https://gitee.com/qc_faith/picture/raw/master/image/202212291650905.png" alt="在这里插入图片描述" style="zoom:50%;" />

执行好applyBeanPostProcessorsAfterInitialization方法后，initializeBean方法的主体逻辑便执行完了，接下来我们继续看doCreateBean的剩余逻辑。
这里的earlySingletonExposure部分逻辑也是用来处理循环依赖的。这部分主要检查：bean在进行属性设置和切面增强后生成的对象和earlySingletonObjects内保存的对象引用是否相同，若不相同则会抛出异常。earlySingletonObjects内保存的对象是在当前bean构建过程中提前暴露出的“半成品”对象引用。

<img src="https://gitee.com/qc_faith/picture/raw/master/image/202212291629450.png" alt="在这里插入图片描述" style="zoom:50%;" />

### registerDisposableBeanIfNecessary

在进行好依赖注入及各种操作后，容器会检查singleton类型的bean实例，看其

（1）是否实现了DisposableBean接口、

（2）是否实现了destory-method、

（3）是否利用@PreDestroy标注了销毁前需要执行的方法。

如果满足任意条件，就会为该bean实例注册一个用于对象销毁的回调，以便在这些singleton类型的对象实例在销毁之前，执行销毁逻辑。

```java
/**
	 * Add the given bean to the list of disposable beans in this factory,
	 * registering its DisposableBean interface and/or the given destroy method
	 * to be called on factory shutdown (if applicable). Only applies to singletons.
	 * @param beanName the name of the bean
	 * @param bean the bean instance
	 * @param mbd the bean definition for the bean
	 * @see RootBeanDefinition#isSingleton
	 * @see RootBeanDefinition#getDependsOn
	 * @see #registerDisposableBean
	 * @see #registerDependentBean
	 */
	protected void registerDisposableBeanIfNecessary(String beanName, Object bean, RootBeanDefinition mbd) {
		AccessControlContext acc = (System.getSecurityManager() != null ? getAccessControlContext() : null);
		if (!mbd.isPrototype() && requiresDestruction(bean, mbd)) {
			if (mbd.isSingleton()) {
				// Register a DisposableBean implementation that performs all destruction
				// work for the given bean: DestructionAwareBeanPostProcessors,
				// DisposableBean interface, custom destroy method.
				registerDisposableBean(beanName,
						new DisposableBeanAdapter(bean, beanName, mbd, getBeanPostProcessors(), acc));
			}
			else {
				// A bean with a custom scope...
				Scope scope = this.scopes.get(mbd.getScope());
				if (scope == null) {
					throw new IllegalStateException("No Scope registered for scope name '" + mbd.getScope() + "'");
				}
				scope.registerDestructionCallback(beanName,
						new DisposableBeanAdapter(bean, beanName, mbd, getBeanPostProcessors(), acc));
			}
		}
	}
```

这些自定义的对象销毁逻辑，在对象实例初始化完成，并注册了相关的回调接口后，并不会马上执行。回调方法注册后，返回的对象即处于“可用”状态，只有该对象不再被使用的时候，才会执行相关的自定义销毁逻辑，此时通常也就是Spring容器关闭的时候。在ApplicationContext容器中销毁接口的回调主要是通过JVM的shutdownHook实现的，ApplicationContext容器通过registerShutdownHook()方法注册JVM关闭的回调钩子，保证在JVM退出前，这些singleton类型bean对象的自定义销毁逻辑会被执行。下面是对象销毁时销毁回调接口的执行逻辑。到此为止整个doCreateBean()函数就执行完了，一个完整可用的Bean对象也被构建出来了。

<img src="https://gitee.com/qc_faith/picture/raw/master/image/202212291629523.png" alt="在这里插入图片描述" style="zoom:50%;" />

## Spring对循环依赖的处理

如果大家能够一直读到这里的话，会发现整个Bean的构建流程中穿插了很多Spring循环依赖的处理逻辑，我们在文章末尾总结下Spring循环依赖问题。Spring 循环依赖有以下几种情况：

1. **prototype类型bean不支持循环依赖，若出现循环依赖会抛出异常。**

   prototype类型bean的生命周期不由spring容器管理，属于即产即用型，所以Spring并不会持有bean的引用，更不会有bean的缓存，所以无法实现循环依赖（循环依赖的实现需要缓存bean）。

2. **singleton类型bean使用构造器注入时，不支持循环依赖，若出现循环依赖会抛出异常。**

   使用构造器注入时，对象会在实例化阶段选定合适的构造器，然后从容器获取构造器的所有参数对象，使用的也是getBean()函数。假如对象A和对象B都使用构造器注入且存在互相依赖，那么当A在实例化时发现自己需要对象B，就调用getBean(B)获取对象B。这时B开始构建且在实例化时发现自己需要对象A，同样去调用getBean(A)获取对象A。这样就导致了死循环，对象A和对象B都没办法实例化，他们连个“半成品”对象都还没生产出来，就阻塞在实例化阶段了。

3. **singleton类型bean使用field注入(即@Autowired或@Resource注入)时，在没有动态代理时支持循环依赖。当进行动态代理时分为两种情况：（1）使用普通切面生成代理支持循环依赖、（2）使用@Async等注解生成代理则不支持循坏依赖。**

   其实这个主要是因为两者生成动态代理的逻辑有所不同。在介绍具体原因前，我们需要先了解下Spring中用于处理循环依赖的“三级缓存”。

> 【spring循环依赖】三个重要的Map(“三级缓存”)
>
> 1. **singletonObjects：缓存所有构建好的bean**
> 2. **earlySingletonObjects：在bean构建过程中提前暴露出来的“半成品”bean**
> 3. **singletonFactories：用于创建“半成品”bean的工厂**

> **（1）singletonObjects**：singletonObjects中存储的都是完全构建好的bean，容器启动后在项目代码中我们通过getBean()从容器中获取bean实例时，就是直接从该缓存中获取，所以效率非常高，同时性能非常好。
> **（2）earlySingletonObjects**：earlySingletonObjects主要保存在对象构建过程中提前暴露出来的“半成品”bean，是解决大部分循环依赖的关键。假设有两个Bean A和B，这两个Bean间存在相互依赖，且都不存在动态代理。那么当容器构建A时，实例化好A后，容器会把尚未注入依赖的A暂时放入earlySingletonObjects中，然后去帮A注入依赖。这时容器发现A依赖于B，但是B还没有构建好，那么就会先去构建B。容器在实例化好B时，同样需要帮B注入依赖，此时B发现自己依赖A，在获取A时就可以从earlySingletonObjects中获取尚未注入依赖的A引用，这样就不会阻塞B的构建，B完成构建后就又可以注入到A中，这样就解决了简单的循环依赖问题。
> **（3）singletonFactories**：singletonFactories用来保存创建“半成品”bean的工厂实例，在Bean实例化好后，并不会直接将尚未注入依赖的bean直接放入到earlySingletonObjects中，而是将能够创建该“半成品”bean的工厂实例放入到singletonFactories中。这样我们在对外暴露“半成品”bean时就可以加入一些自定义的特殊处理逻辑。例如下图中普通切面代理就会在此处动些“手脚”。

有了上面的基础知识后，我们就可以分析为什么在使用@Async注解时无法支持循环依赖了。
假设对象A使用@Aysnc注解且和对象B存在循环依赖：对象A实例化后进行依赖注入时，发现自己依赖对象B，就会通过getBean(B)获取对象B。由于对象B尚未构建，就开始构建对象B，对象B实例化后注入依赖时发现依赖对象A，同样通过getBean(A)获取对象A。**这时返回给对象B的是尚未注入依赖且还没有代理的对象A。** 对象B在构建完成会将引用返回给对象A，对象A继续进行构建，在完成所有的依赖注入且执行完初始化方法后，会执行动态代理逻辑，这时@Aysnc代理实现类AsyncAnnotationBeanPostProcessor **为对象A生成了代理对象**并将代理对象作为最终对象返回。这时就会**导致最终生成的【代理后对象A】和对象B依赖的【原生对象A】并不是同一个对象**，这种情况下Spring会抛出异常。
由于普通切面代理实现类AnnotationAwareAspectJAutoProxyCreator做了些特殊的处理，所以不会出现上述问题。具体实现逻辑可以参照下面的时序图。
<img src="https://gitee.com/qc_faith/picture/raw/master/image/202212291649734.png" alt="在这里插入图片描述" style="zoom:50%;" />