# 架构知识

> 其实组件不止下述这些，个人编写简易Spring以及简易SpringMvc项目中略过了一些非必要但也很重要的模块。想快速了解可以看看[Spring FrameWork 5](https://pdai.tech/md/spring/spring-x-framework-introduce.html)专栏，鱼皮大大写的。想深入了解的话推荐看陈雄华老师在电子工业出版社出版的Spring有关书籍，讲的很细，日后希望可以详读。

## Core Container

> 实现其他模块的基础，由Beans模块、Core核心模块、Context上下文模块和SpEL表达式语言模块组成。想在项目中实现AOP等功能必须仰赖于该模块。下面介绍一下SpEL模块外的三个模块

- **Beans 模块**：提供了框架的基础部分，包括控制反转和依赖注入。
- **Core 核心模块**：封装了 Spring 框架的底层部分，包括资源访问、类型转换及一些常用工具类。
- **Context 上下文模块**：建立在 Core 和 Beans 模块的基础之上，集成 Beans 模块功能并添加资源绑定、数据验证、国际化、Java EE 支持、容器生命周期、事件传播等。ApplicationContext 接口是上下文模块的焦点。

## Aspects

> 提供与 AspectJ 的集成，是一个功能强大且成熟的面向切面编程（AOP）框架。（和Spring AOP实现上没有关系，只是Spring可以集成AspectJ，使用相关方法）

## IOC

> 控制反转，反转的是依赖对象的获取。由容器创建对象并且进行管理（在合适的时候注入），取代了本来由我们自己创建获取对象的操作。在Spring Boot中使用@Autowired注解就搞定对象注入，在项目中得深入看看背后的过程了。

### 配置方式

1. xml配置：挺麻烦的，创建xml文件声明命名空间和配置bean，bean多的话能配十万八千行，但是适用于任何场景
2. Java配置：声明一个Config类，注解`@Configuration`，在方法中注解@Bean创建对应实例并返回。同样适用于任何场景，但是大量配置可读性就比较差
3. **注解配置：**简单易用，打个注解就行，但是较之于前两种方法，不能适用于所有场景，通常情况下采用这种方法

### 依赖注入方式

1. 构造方法注入：正如其名，在调用构造函数时，实现依赖对象的注入
2. setter注入：正如其名，使用set方法实现依赖对象注入
3. 注解注入：使用`@Autowired`注解进行注入

官方推荐构造方法注入，既保证注入组件不可变，又确保需要的依赖不为空，这里就得看下具体的注入方法了

```java
@Service
public class ServiceImpl {

    private final DaoImpl daoImpl;

    public ServiceImpl (final DaoImpl daoImpl) {
        this.daoImpl = daoImpl;
    }
}
```

1. final保证依赖对象是不可变的
2. 由于实现有参构造函数，就必须传入对应参数才能构造对象
3. 可以获取原始对象，即构造方法创建出来的对象
4. 可以解决循环依赖问题，采用构造方法注入时。spring项目启动会抛出：BeanCurrentlyInCreationException：Requested bean is currently in creation: Is there an unresolvable circular reference？从而提醒你避免循环依赖

### 整体功能

> 总体来说功能如下图所示嘞：

![IOC功能](笔记.assets\spring-framework-ioc-source-7.png)

1. 加载Bean配置：对不同类型资源的加载（比如xml配置文件），解析成生成统一Bean的配置（Bean容器，对Bean进行定义和行为管理）
2. 根据Bean定义加载生成Bean实例，并放置在Bean容器中：比如Bean依赖注入、Bean嵌套、Bean缓存
3. 特殊Bean的处理：比如国际化Message等生成特殊类结构支撑
4. 对容器中Bean提供统一管理和调用：一般来说采用工厂模式

### 体系结构

> 主要分为三大块！：
> 
> 1. BeanFactory和BeanRegistry：定义IOC容器功能规范和Bean的注册
> 2. BeanDefinition：定义各种Bean对象及其相互的关系
> 3. ApplicationContext：IOC容器的接口类，在基本IoC容器实现基础上实现访问资源、国际化、应用事件等功能

#### BeanFactory

> 列出实现的方法如下：

1. 根据bean名字和Class类型来获得bean实例
2. 返回指定bean的Provider
3. 检查工厂中是否包含给定name的bean，或者外部注册的bean
4. 检查所给定name的bean是否为单例/或原型
5. 判断所给name的类型与type是否匹配
6. 获取给定name的bean的类型
7. 返回给定name的bean的别名（Aliases）

#### BeanRegistry

> Spring配置文件中的每一个`bean`在Spring容器中都会通过BeanDefinition对象表示。BeanDefinitionRegistry接口提供向容器手工注册BeanDefinition对象的方法

#### BeanDefinition

> 对Bean对象和对象间关系进行定义，相关的还有BeanDefinitionReader（BeanDefinition的解析器，完成对Spring配置文件的解析），BeanDefinitionHolder（eanDefination的包装类，用来存储BeanDefinition，name以及aliases等）

#### ApplicationContext

> 继承自BeanFactory，实现定义基础Bean容器功能基础上额外拓展处理特殊Bean的能力，内含Bean工厂、应用事件、资源加载、国际化相关接口，有众多的扩展实现。

#### 总结

> 分析完之后，鱼皮大大的架构图可以对应相关模块如下

![spring-framework-ioc-source-71](笔记.assets\spring-framework-ioc-source-71.png)

个人认为，BeanFactory面向的是Spring开发者，而ApplicationContext面向的是Spring使用者。这也是封装的含义所在 

### 初始化流程

Spring IoC容器载入定义资源从refresh()函数开始

> Spring实现将资源配置通过加载、解析、生成BeanDefinition注册到IoC容器（**本质上是存放BeanDefinition的ConcurrentHashMap<String,Object>**）的过程

**从源码上看，具体实现为**

1. 调用父类构造方法为容器设置好Bean资源加载器：
   
   1. 调用默认构造函数初始化容器id，name，状态以及资源解析器
   2. 调用setParent方法将父容器的Environment合并到当前容器

2. 设置配置路径进行Bean定义资源文件定位
   
   1. 调用setConfigLocations方法从Environment中解析Bean配置文件路径

3. 初始化容器，调用模板方法refresh（该模板提供钩子方法）
   
   1. 创建IoC容器前，如果有容器存在，需要将已有容器销毁和关闭
   2. 建立新的Ioc容器

**从流程上看：**

![spring-framework-ioc-source-9](笔记.assets\spring-framework-ioc-source-9.png)

1. 调用refresh方法进入初始化入口
2. 调用loadBeanDefinition方法载入beanDefinition
   1. 通过ResourceLoader实现资源文件位置定位
   2. 通过BeanDefinitionReader完成定义信息的解析和Bean信息的注册
   3. 实现BeanDefinitionRegistry接口将BeanDefinition注册到IoC容器中
3. 通过BeanFactory和ApplicationContext使用Spring的IoC服务

> 这里补充一下关于模板方法模式的一些知识
> 
> **模板方法是一个算法骨架，是一系列调用的基本方法，而基本方法可以分为：**
> 
> 1. 抽象方法：声明但未实现
> 2. 具体方法：实现，可以重写或继承
> 3. 钩子方法：实现，分为用于判断的逻辑方法，和需要子类重写的空方法两种
> 
> **在IoC容器初始化过程中用到的钩子方法有**
> 
> 1. prepareRefresh：对应BeanFactory、MessageSource、ApplicationEvent的初始化
> 2. onRefresh：注册监听器
> 3. finishRefresh：ioc容器初始化完成
> 4. cancelRefresh：销毁已初始化的ioc容器（这个方法是前序方法出错调用的

### Bean实例化

#### getBean(String name)

> 具体流程体现在**getBean**方法，注意的是Spring进行Bean实例化时：会确保依赖也被初始化。根据beanDefinition的信息（单例、原型、bean的scope）有三种不同的创建bean实例方法。

**大概的流程是**：

1. 从beanDefinitionMap通过beanName获得BeanDefinition
2. 从BeanDefinition中获得beanClassName
3. 通过反射初始化beanClassName的实例instance
   1. 构造函数从BeanDefinition的getConstructorArgumentValues()方法获取
   2. 属性值从BeanDefinition的getPropertyValues()方法获取
4. 返回beanName的实例instance

> 上面提到了依赖，这个依赖我们不难想象。也许beanA和beanB存在这样的关系，beanA中有属性beanB，而beanB中也有属性beanA。正常程序中是完全不会有影响的，因为并不会强制要求属性与对象同时进行初始化。但是上文步骤4中提到，Spring源码中会要求依赖也被确保初始化！wow，here is the problem，come to solve it！

#### 三级缓存

> 三级缓存是Spring为解决单例bean的循环问题而使用的一种策略。具体而言体现在调用**getSingleton**方法，会查找bean是否在缓存中，在哪一级，以及进行相对应的处理

**哪三层？**

1. 一级缓存（singletonObjects）：单例对象缓存池的成熟对象，已经实例化并且属性赋值
2. 二级缓存（earlySingletonObjects）：单例对象缓存池的半成品对象，已经实例化但属性尚未赋值
3. 三级缓存（singletonFactories）：单例工厂的缓存

**方法执行过程？**

1. 先从一级缓存singletonObjects中获取
2. 若是获取不到，而且对象正在建立中，再从二级换从earlySingletonObjects中获取
3. 若是仍然获取不到且容许singletonFactories经过get

**三级缓存解决循环依赖的整个流程解释：**

1. A实例化并放入三级缓存中，注意此时放入的是getEarlyBeanReference得到的对象，如果该对象需要被代理，那么此时已经生产代理对象
2. B实例化并在三级缓存中找到A对象，此时将A对象升入二级缓存中
3. B初始化完成，进入一级缓存
4. A初始化完成，进入一级缓存

**解决不了哪些问题？**

1. 构造器的循环依赖：调用构造方法之前无法将对象放入三级缓存中，策略失效
2. protype作用域循环依赖：Spring不缓存protype作用域bean，策略失效
3. 多例的循环依赖：多实例bean每调用一次getBean都会执行一次构造方法，没有三级缓存

> 这里提一下上述对象怎么解决：
> 
> 1. 生成代理对象产生的循环依赖可以采用延迟加载，或指定加载先后顺序解决
> 2. 使用@DependsOn产生的循环依赖，需要找到对应地方迫使其不再循环依赖
> 3. 多例产生循环依赖：将bean改为单例
> 4. 构造器循环依赖：采用延迟加载解决

### Bean生命周期

> 从逻辑上分，可以分为实例化、属性赋值、初始化、销毁几个部分
> 
> 从方法调用而言，涉及到BeanPostProcessor（容器级生命周期接口方法）、Bean自身的方法、Aware接口相关方法（Bean级生命周期接口方法），Spring通过独立于Bean的接口实现类可以实现对Bean生命周期做动态增强的功能

**流程大致如下：**

![Bean生命周期](笔记.assets\Bean生命周期.png)

**BeanFactoryProcessor&BeanPostProcessor**

1. BeanFactoryPostProcessor：spring提供的容器扩展机制，允许我们在bean实例化之前修改bean的定义信息即BeanDefinition的信息。其重要的实现类有PropertyPlaceholderConfigurer和CustomEditorConfigurer，PropertyPlaceholderConfigurer的作用是用properties文件的配置值替换xml文件中的占位符，CustomEditorConfigurer的作用是实现类型转换

2. BeanPostProcessor：spring提供的容器扩展机制，不同于BeanFactoryPostProcessor的是，BeanPostProcessor在bean实例化后修改bean或替换bean。（实例化后再工作，很容易联想到AOP吧！）

## AOP

> 提供了面向切面编程实现，提供比如日志记录、权限控制、性能统计等通用功能和业务逻辑分离的技术，并且能动态的把这些功能添加到需要的代码中，这样各司其职，降低业务逻辑和通用功能的耦合。

简单来说，Spring实现AOP通过动态代理。而动态代理呢，又可以分为JDK、Cglib几种

JDK的动态代理的缺点是只能对接口作代理，其原理是反射。

而Cglib则可以对任何类进行代理，其原理是字节码增强，更灵活更高效，但是会更复杂。（不过Spring屏蔽了复杂性，只是原理复杂

# 实现思路

> 整个项目的切入点都是从`getBean()`开始的，在获取Bean的时候需要执行实例化、属性填充、初始化操作，不再使用Bean时需要执行销毁操作。这些操作是通过什么实现的？此外Bean是如何存放在容器中的？容器是怎么样的？如何实现对Bean功能的动态增强(AOP)等等就在下文慢慢说道

## IOC

### 1.实现简易的Bean容器

要实现getBean方法，没有容器是万万不能的

> Spring包含并管理应用对象的配置和生命周期，Bean通过零件化拆分存放至BeanDefinitoin的方式，让Spring更加容易管理Bean对象，并可以统一进行装配(包括Bean初始化、属性填充等)

**如何实现一个Spring容器**(BeanFactory)

简单的方法就是用Map，可以用HashMap也可以用ConcurrentHashMap。项目中采用HashMap，实质上就是存储Bean名称和BeanDefinition的映射

### 2.实现Bean的定义、注册、获取

定义：BeanDefinition，存储Bean类信息

注册：将定义存储在容器中

获取：通过单例模式获取Bean的实例(需要判断bean是否在缓存中，并根据相关情况进行实例化操作)

**模板模式**：

Bean工厂提供Bean获取方法，并由AbstratBeanFactory抽象类进行实现。统一接口的通用核心方法的调用逻辑和标准定义！类的继承者只需要关心具体方法的逻辑实现

### 3.提供对象实例化策略

如果是对于没有属性的bean来说(大部分常见bean都是无参的)，通过beanClass.newInstance就已经完成任务了，但是对于有属性的bean而言，实例化分成两种流派，各有千秋

1. 无参构造：不用获取Constructor，直接创建实例，对于jdk而言是newInstance，对于cglib而言是enhancer.create
2. 有参构造：需要获取对应的Constructor，通过对应的构造器来进行实例化

#### 构造函数

需要获取bean的Constructor，并与实例化所给的参数进行匹配，在Spring中会检查参数的类型和个数是否都匹配，但是为了简单项目中只会检查参数的个数是否匹配

#### NoOp vs MethodInterceptor

> 一开始用MethodInterceptor作为enhancer的callback一直报错，用了NoOp才解决，稍微探究一番

callback可以认为是cglib生成字节码的实现手段，一共可以分为6种

1. MethodInterceptor：类似于环绕增强
2. FixedValue：替换源方法的返回值
3. InvocationHandler：用于自定义实现，类似于MethodInterceptor
4. LazyLoader：单例类的延迟生成代理
5. Dispatcher：原型类的延迟生成代理
6. NoOp：啥也不干

实现MethodInterceptor接口时，需要实现intercept方法，参数为对象、方法、方法参数、方法代理

### 4.属性填充、依赖注入

> 这里的属性更偏向于成员变量的意思，实质上属性的官方定义是指get或set方法去掉get或set后剩下字母。比如 getSize 即有属性size

**实现**：在BeanDefinition中添加属性相关内容，以List+Map形式存储bean类的成员变量名称和类型。进行属性填充即通过BeanDefinitoin中的属性信息与bean类的属性信息进行匹配，将BeanDefinition存储的属性填充进bean实例中

**注意事项**：进行属性填充时需要注意该属性是否为对象类型，单独对对象的依赖进行管理。在注入对象依赖时，需要先对注入的对象进行实例化

### 5.资源加载器解析文件注册对象

> 一般来说Spring有配置文件就可以实例化bean，不用手动创建。所以我们需要一个资源解析器，可以读取classpath、本地文件、云文件中的配置内容(xml)，并且可以对配置文件进行解析，将Bean对象注册到Spring容器中

**实现**：首先是定义资源和资源解析器。具体来说实现有classpath资源、文件系统资源和Url资源，资源解析器的默认实现负责读取这三种资源。随后实现对xml文件的解析，主要是根据读取标签相关值，根据标签名进行相应处理，生成bean元信息(beanDefinition)

#### org.w3c.dom

org.w3c.dom为DOM(文档对象模型)提供接口

Document(文档树模型)则可以表示整个HTML或XML文档

Node是DOM中的基本对象，代表文档树的抽象节点(标签啦)

Element是Node类最主要的子对象，可以存取属性

#### CharSequence

> 使用hutool的StrUtil建议使用CharSequenceUtil，有啥不一样呢？

CharSequence是一个接口，表示有序的字符集合，提供一些基本的操作方法

String、StringBuilder、StringBuffer都实现该接口

**较之于String**：可读可写；泛用性更强

### 6.应用上下文：超级BeanFactory

> ApplicationContext：把相应的XML加载、注册、实例化以及新增的修改和扩展都直接实现
>
> 方便、快捷，是Spring的对外接口

**refresh方法**:

1. 加载：创建BeanFactory，从Xml文件加载BeanDefinition，获取BeanFactory
2. 修改：Bean实例化前，可以进行BeanDefintion的修改
3. 注册：BeanPostProcessor提前于其他对象实例化之前执行注册操作
4. 实例化：Bean对象(BeanPostProcessor)
	1. 前置处理
	2. CreateBean
	3. 后置处理

**BeanFactoryPostProcessor**：

实现在所有BeanDefinition加载完成后，实例化Bean对象之前，提供修改BeanDefinition属性的机制

**BeanPostProcessor**：

修改实例化Bean对象的扩展点，可以在Bean对象执行初始化方法前后执行相关方法

### 7.初始化和销毁方法

> Spring中bean可以读取在xml文件中自定义的初始化和销毁方法名，调用实现接口的对应方法

**InitalizingBean、DisposableBean**：

定义初始化方法和销毁方法的接口，想实现初始化和销毁方法的bean可以实现这两个接口

当然，并不是所有的初始化和销毁方法都需要自己定义去实现，肯定是需要进行一些初步的具体实现的。DisposableBean的初步实现就由DisposableBeanAdapter适配器完成，这里涉及到了适配器模式，让方法有了更多的延展性

### 8.Aware感知容器对象

> 获得Spring框架提供的BeanFactory、ApplicationContext、BeanClassLoader等等进行框架扩展使用时，涉及到对容器操作的感知

#### Aware

> 感知标记性接口，具体子类定义和实现可以感知容器中相关对象

实现BeanFactory、BeanClassLoader、BeanName、ApplicationContext的感知。其中由于ApplicationContext的相关属性在AbstractBeanAutowireCapableBeanFactory无法直接感知，所以通过refresh方法写入BeanPostProcessor再进行调用

### 9.FactoryBean

> 提供把复杂的且以代理方式动态变化的对象注册到Spring容器中的功能

`FactoryBean`接口提供**getObject**方法获取对象，实现此接口的对象类可以扩充自己的对象功能。以MyBatis为例，其实现了MapperFactoryBean类，在getObject方法中提供SqlSession执行CRUD方法

`FactoryBeanRegistrySupport`定义从缓存中获取FactoryBean，和获取单例、原型FactoryBean的方法，在AbstractBeanFactory的doGetBean方法中引入该类的方法以获取FactoryBean的实例

#### InvocationHandler&Proxy

`InvocationHandler`是proxy代理实例的调用处理程序实现的接口，**当通过动态代理对象调用一个方法时，方法的调用会被转发到实现InvocationHandler接口类的invoke方法进行调用**

`Proxy`是创建一个代理对象的类，最常用的方法是**newProxyInstance**：创建一个代理类对象

### 10.Event

> 对一些特殊的事件比如初始化和销毁、或者一些用户自定义的事件，可以进行监听，完成一些自定义的动作。

借助于Java中的Event机制实现，有点类似于C语言的回调函数

在Context中加入event包，提供相关的事件处理接口、实现类，自定义事件等

### IOC总结

> 知识感想

在整个实现简易Spring IoC过程中学习到相当多知识点，比如`ApplicationContext`实现`close`方法时使用的`Hook`；进行JSON序列化、Assert断言、字符串判断、反射设置属性值使用的`HuTool`；实现动态实例化使用的`reflect`、`Cglib`；实现事件相关机制使用的`Event`；读取xml文件使用的`org.w3c.dom`；`BeanFactory`实现的`简单工厂模式`、`FactoryBean`实现的`工厂方法模式`、通过`策略模式`访问资源、通过`观察者模式`实现事件的监听、通过`适配器模式`实现功能的拓展；和贯穿整个bean生命周期的`refresh`方法更是对`模板方法`的透彻使用；`单例模式`确保获取同一对象的`getSingelton`方法等

> 流程解析

在这里对IOC部分实现做出一点总结，一切的一切还得从getBean开始说起

```java
/**
 * Bean实例化的方法
 * @param beanName bean名
 * @return 返回bean实例
 * @throws BeansException bean相关异常
 */
Object getBean(String beanName) throws BeansException;

//上面是定义，下面是实现，不在一个类/接口中
@Override
public Object getBean(String beanName) throws BeansException {
    return doGetBean(beanName, null);
}
public Object doGetBean(final String beanName, @Nullable final Object[] args) {
        Object sharedInstance = getSingleton(beanName);
        // 缓存中有bean，或者说bean已创建
        if (sharedInstance != null) {
            return getObjectForBeanInstance(sharedInstance, beanName);
        }
        BeanDefinition beanDefinition = getBeanDefinition(beanName);
        Object bean = createBean(beanName, beanDefinition, args);
        // 调用此方法以区分获取FactoryBean和其他Bean
        return getObjectForBeanInstance(bean, beanName);
    }
```

在`BeanFactory`接口中，定义getBean方法，可以通过beanName来获取bean，那必然存在一种映射关系可以使得beanName来指向bean实例。说到映射，很容易就想到通过map来实现，那么引入一个问题，如何将bean存入这个map里呢。

所以这里引入BeanRegistry接口的概念，注意除单例模式外的bean都不由Spring容器进行生命周期的管理，像原型之类的创建了可能就直接丢给用户了，不会在管了，所以需要存到map中的bean其实只有单例bean。在`SingletonBeanRegistry`接口中定义获取map中单例bean和向map中注册单例bean的方法

回到getBean，现在已经拥有了获取已经存在的单例bean的方法，那getBean还需要考虑另一件事，如何创建一个bean，或者说创建一个Bean需要哪些信息？这里引出`BeanDefintion`的概念，对于bean而言，或者说java类而言，最直接的方法无非是直接使用Class.newInstance进行实例化。自然而然的beanDefinition中需要存储beanClass的信息，当然随着后续的扩展，在属性填充时引入了属性键值对的概念；在初始化-销毁过程中由`BeanPostProcessor`进行功能扩展时引入了初始化和销毁方法的概念；在区分单例和原型时引入了scope的概念

话说回来，有了以上的信息又要怎样去创建一个bean呢，这里我们进入createBean方法来看看！

```java
/**
 * 缓存或者说实例化bean
 * @param beanName bean的名字
 * @param beanDefinition  bean的元信息
 * @param args bean构造函数的参数
 * @return 创建后的bean
 * @throws BeansException Bean相关异常
 */
protected abstract Object createBean(String beanName, BeanDefinition beanDefinition, @Nullable Object[] args) throws BeansException;

// 上面是定义，下面是实现，不在同一个类/接口中
@Override
    protected Object createBean(String beanName, BeanDefinition beanDefinition, @Nullable Object[] args) throws BeansException {
        return doCreateBean(beanName, beanDefinition, args);
    }

    protected Object doCreateBean(String beanName, BeanDefinition beanDefinition, @Nullable Object[] args) throws BeansException {
        Object bean;
        try {
            // 构造方法参数为空就直接实例化，不为空就找到对应构造器进行实例化
            bean = instantiationStrategy.instantiate(beanDefinition, args);
        } catch (Exception e) {
            throw new BeansException("Instantiation of bean failed", e);
        }
        // 为bean填充属性
        applyPropertyValues(beanName, bean, beanDefinition);
        // 初始化
        bean = initializeBean(beanName, bean, beanDefinition);
        // 注册实现DisposableBean接口的Bean对象
        registerDisposableBeanIfNecessary(beanName, bean, beanDefinition);
        // 判断是否为单例Bean
        if (beanDefinition.isSingleton()) {
            // addSingleton是要放入缓存里
            registerSingleton(beanName, bean);
        }
        return bean;
    }
```

1. 创建对象：对应bean生命周期的实例化过程，首先获取beanClass，`InstantiationStrategy`下定义实例化的方法instantiate，对应有通过jdk、cglib两种派别的方法。考虑到实例化进行无参构造和有参构造的区别，可以让用户自行决定传入的参数，在方法内对构造器进行匹配
2. 填充属性：对应bean生命周期的属性填充阶段，首先需要定义好存储属性类`PropertyValue`，其名为属性的名，其值为属性的值。然后肯定要读取属性，在Spring中bean属性写在xml文件中很常见，需要实现加载xml文件路径和从xml文件读取相关信息并给beanDefinition设置属性相关值的一系列流程。实现`core.io`包，来完成加载文件路径读取文件信息的功能。通过`XmlBeanDefinitionReader`集百家之长来执行具体方法！
3. 初始化：在属性值填充之后，一个Bean对象可以说已经接近创建好啦！那么缺失的是什么呢？在Spring中可以调用自定义的初始化方法，通过xml文件可以配置初始化方法的定义路径，读取后进行执行。此外通过`BeanPostProcessor`在初始化方法执行前后都可以执行一些自定义的方法，实现功能的增强。(这里提一嘴，实例化之前可以通过`BeanFactoryPostProcessor`进行一些BeanDefinition的修改)
4. 使用：getBean方法会返回bean实例，正常使用bean实例，这个是使用者自己决定的，和Spring没啥关系
5. 销毁：之前在初始化中用到的方法，还是一样一样，对销毁方法也有用。

上面的内容都是基于Spring内部的，要给用户提供相应的功能，仅仅如此显然不行的。这里引入application context上下文的概念，可以看作特殊的bean工厂，实现refresh方法。在代码中引入了Event机制实现了事件的监听，并且引入Aware接口实现对容器对象的感知

## AOP

### 0.前置知识

> 不同于之前ioc容器，aop的实现仰赖于aopalliance等等。有一些概念是无法绕过、十分核心的，这部分内容的具体技术我会写在后面的相关技术模块中

总的来说AOP分为配置和织入两部分工作，配置有由Aspect实现，织入即Weave，简单介绍一下

**程序运行时会调用很多方法，调用的很多方法就叫做Join points（连接点，可以被选择来进行增强的方法点），在方法的前或者后选择一个地方来切入，切入的的地方就叫做Pointcut（切入点，选择增强的方法），然后把要增强的功能（Advice）加入到切入点所在的位置。Advice和Pointcut组成一个切面（Aspect）**

1. Aspect(切面)：由Advice与Pointcut组成，实现功能增强
	1. Advice(通知)：定义切点指定的逻辑，也就是增强的功能，比如打印日志啥的
	2. Pointcut(切入点)：在程序的执行流(也就是代码运行过程中)中，可以选择要切入的方法点
	3. JoinPoint(连接点/切点)：可以被选择来增强的方法
2. Introduction(引入)：添加新的方法、属性到已存在的类中
3. Weaving(织入)：不改变袁磊代码，加入功能增强

### 1.初识AOP切面

> aop动态切面是基于jdk、cglib动态代理进行实现，所以要先实现基本的动态代理相关功能

要进行代理，需要匹配类和方法，需要jdk、cglib进行动态代理生成代理对象进行方法的执行。不难看出需要三个核心要素：

1. 被代理的目标对象：原对象，通过其获取类信息、接口信息等
2. 方法匹配器：对方法进行匹配，这个是通过切点表达式综合aspectj包进行实现的
3. 方法拦截器：如果要在原有方法基础上进行增强，需要通过MethodInterceptor接口实现。在调用原方法的基础上，会执行增强方法的逻辑，在方法前后增加代理行为

简单介绍一下用到的技术

#### aopalliance

AOP联盟的API包，包含针对面向切面的接口，通过它可以实现动态织入的功能。实现MethodInterceptor接口可以在调用方法之前、过程中、之后对方法进行控制，其inoke方法参数为MethodInvocation，是一个和方法有关的执行器。

#### 切点表达式

连接点是程序执行的一个步骤，在AOP中一个连接点会对应一个方法的执行。切点指匹配连接点，切点表达式就是描述与连接点进行匹配的动作，举execution函数为例：

```java
"execution(* cn.bugstack.springframework.test.bean.IUserService.*(..))"
```

### 2.Bean生命周期中实现AOP

> 通过BeanPostProcessor将AOP方法引入Bean生命周期

1. 通过BeanPostProcessor修改Bean对象的扩展信息，将切面的代理对象实例化。
2. 代理对象的创建需要提前于其他对象的创建，需要提前进行判断
3. 方法拦截器的功能需要进一步抽象、整合，方便代理工厂进行调用

### AOP总结

> 知识感想

AOP的核心技术实现就是动态代理的使用，实现动态代理可以通过jdk和cglib的方式，二者区别如下

**原理**：使用jdk代理是运行时增强，通过反射机制生成实现代理接口的匿名类，调用具体方法前调用InvokeHandler处理；使用cglib代理是编译时增强，通过使用字节码处理框架ASM，将代理对象类的class文件加载并修改字节码生成子类

AspectJ是Java中流行的AOP编程扩展框架，换言之也就是AOP的Java实现版本，定义了AOP的语法。引入了join point、pointcut、advic等概念。pointcut和advice一同组成AspectJ的动态部分，**负责指定在什么条件下切断执行，采取什么动作实现切面操作**。(感觉和打断点很类似，运行到pointcut的时候，暂停执行原方法，执行advice定义的行为后再操作)。

> 流程解析

Aop这一块是实现逻辑比较简单，整体就两步

1. 通过aopalliance实现AOP的基本概念，比如Advice、Pointcut
2. 将Aop的相关概念融合进Bean生命周期，比方说bean初始化时要创建好代理对象等

> 感觉这一段比IOC思路上简单多了，哈哈哈哈哈哈

## 优化篇

### 1.实现注解，简化Bean对象配置

> bean配置只能通过xml！实在是太low啦！我的注解在哪里！
>
> 通过包的扫描注册、注解配置使用、占位符属性填充实现自动化bean配置

这一阶段需要分为三步走

#### 1.1.自动扫描bean对象注册

> 扫描一个Bean对象进行注册，比如@Service、@Component

1. 需要扫描路径，要清楚需要注册的Bean对象在哪里，目前设置通过解析xml文件来获取
2. 需要给要注册的bean对象打注解，目前实现@Component注解和@Scope注解，@Component表示该类是需要被注册的，@Scope则声明了bean的作用域(支持单例和原型)
3. 扫描Class对象、获取Bean元信息，注册Bean对象

#### 1.2.注解注入属性信息

> 扫描Bean对象注册，属性也要可以通过注解填充！属性大体有两种，一种为基本数据类型属性，另一种为对象属性。分别对应@Value和@Autowired注解

1. 扫描注解和过滤字段，这部分和1.1是类似的，不再赘述
2. 在xml文件读取获取相关信息后，在属性填充处理前，通过BeanPostProcessor来将属性值注入bean！
3. 对于@Value注解而言，一般设置属性值的格式为${xxx}，获取字符串xxx，去properties获取对应字符串
4. 对于@Autrowired注解而言，注入对象无非是调用getBean方法，由于传入的参数只有属性值的Class类型，所以需要定义新的getBean方法(主要是改参数啦)

#### 1.3.AOP代理对象的属性注入

> 之前创建代理对象在创建其他Bean对象之前，现在要融入Bean生命周期中，在Bean对象执行初始化方法后，再执行代理对象的创建

1. 判断代理类通过jdk代理创建还是cglib，判断接口时需要进行对应操作。(这一步是对之前的优化)
2. 将创建代理操作后移到执行Bean的初始化方法之后(之前是在实例化之前)

### 2.处理循环依赖

> 项目中还没有解决大名鼎鼎的循环依赖问题，仿照Spring实现三级缓存处理循环依赖

一级缓存可以解决循环依赖吗？一实例化就马上将bean塞入缓存中，将半成品bean与初始化好的bean都放在singleton中进行保存。但是解决不了代理对象的循环依赖（代理对象在初始化方法执行完之后创建，所以在实例化时代理对象是无法获取到代理对象的）

为什么需要二级缓存？区分实例化后和初始化后的Bean！二者是不一样的！是要分开的

**三级缓存**

1. 一级缓存：存放成品bean对象
2. 二级缓存：存放半成品(实例化但未初始化的)Bean对象
3. 三级缓存：存放代理对象(只有代理对象会在三级缓存中)

引入三级缓存，首先要对Singelton相关方法进行更新。重点关注getSingelton方法，如今获取bean不能仅仅局限于第一级缓存，要对每一级缓存都进行考虑！当遇到三级缓存中有代理对象时，需要将代理对象的真实对象获取到，放入二级缓存中！

```java
public Object getSingleton(String beanName) {
        Object singletonObject = singletonObjects.get(beanName);
        if (null == singletonObject) {
            singletonObject = earlySingletonObjects.get(beanName);
            // 判断二级缓存中是否有对象，注意只有代理对象在三级缓存中哦
            if (null == singletonObject) {
                ObjectFactory<?> singletonFactory = singletonFactories.get(beanName);
                if (singletonFactory != null) {
                    singletonObject = singletonFactory.getObject();
                    // 把三级缓存中的代理对象中的真实对象获取出来，放入二级缓存中
                    earlySingletonObjects.put(beanName, singletonObject);
                    singletonFactories.remove(beanName);
                }
            }
        }
        return singletonObject;
    }
```

那么如何通过三级缓存解决Bean代理对象的循环依赖呢？在Spring中会调用 SmartInstantiationAwareBeanPostProcessor#getEarlyBeanReference(..) 方法对提前暴露的Bean进行处理。项目中没有实现该接口，仅仅实现了该方法。

```java
protected Object getEarlyBeanReference(String beanName, BeanDefinition beanDefinition, Object bean) {
    Object exposedObject = bean;
    for (BeanPostProcessor beanPostProcessor : getBeanPostProcessors()) {
        if (beanPostProcessor instanceof InstantiationAwareBeanPostProcessor) {
            exposedObject = ((InstantiationAwareBeanPostProcessor) beanPostProcessor).getEarlyBeanReference(exposedObject, beanName);
            if (null == exposedObject) {
                return null;
            }
        }
    }
    return exposedObject;
}
```

```java
@Override
public Object getEarlyBeanReference(Object bean, String beanName) {
    earlyProxyReferences.add(beanName);
    return wrapIfNecessary(bean, beanName);
}
```

earlyProxyReferences是提前暴露的bean，所有的bean都是会被先放到三级缓存的！

### 3.数据类型转换

> xml文件中默认提供的字符串数据，但是在之前项目中还需要进行手动转换！全自动洗衣机怎么能不全自动，奥力给！

通过FactoryBean实现提供转换对象的服务GenericConversionService，并在createBean中的applyPropertyValues方法中调用属性转换的方法，实现数据类型转换的操作

# 心得体会

## 设计模式

### 模板方法

> 模板方法会定义一个算法的骨架，将一些步骤延迟到子类去实现。使得在子类不改变算法结构的基础上，可以重新定义算法中的某些步骤

**使用**：

使用该设计模式的类中，一般有如下几种方法

1. 抽象方法：抽象类声明，由具体子类实现，并以abstract关键字进行标识
2. 具体方法：抽象类声明并实现，子类不Override
3. 钩子方法：抽象类声明并实现，子类可以Override。通常抽象类会给出没有具体实现的空方法
4. 模板方法：定义算法逻辑骨架的方法

在代码实现中，refresh方法就是一个典型的模板方法

```java
public void refresh() throws BeansException {
    // 1.创建BeanFactory，并加载BeanDefinition
    refreshBeanFactory();
    // 2.获取BeanFactory
    ConfigurableListableBeanFactory beanFactory = getBeanFactory();
    // 3.Bean实例化之前，执行BeanFactoryPostProcessor
    invokeBeanFactoryPostProcessors(beanFactory);
    // 4.BeanPostProcessor 需要提前与其他Bean对象实例化之前进行注册操作
    registerBeanPostProcessors(beanFactory);
    // 5.提前实例化单例对象
    beanFactory.preInstantiateSingletons();
}
```

refreshBeanFactory和getBeanFactory是抽象方法

```java
protected abstract void refreshBeanFactory() throws BeansException;

protected abstract ConfigurableListableBeanFactory getBeanFactory();
```

invokeBeanFactoryPostProcessors和registerBeanPostProcessors是具体方法

**优点/应用场景**：

个人认为使用该设计模式可以在以达成某一特定目标为前提时，可以在一个整体的框架之下，有不同具体的实现方式。不仅

### 简单工厂/工厂模式

> 通过工厂对对象进行管理，重点在于如何维护对象的生命周期

简单工厂和工厂模式的区别在于：

1. 工厂模式的每个具体产品都有对应的工厂类

BeanFactory会根据传入标识来获得Bean对象，体现简单工厂

FactoryBean让实现接口的Bean获取自己创建对象的能力，体现工厂模式

### 代理模式

> 代理可以分为静态代理和动态代理，静态代理只需要简单的实现接口进行转换即可，而动态代理可以哦通过JDK、Cglib实现

1. jdk：通过反射实现，代理类需要继承Proxy，故而只能对接口进行代理
2. cglib：通过ASM编辑字节码实现，代理类直接继承被代理类

### 单例模式

> 保证每次获取的都是相同的对象

比较经典的是双重校验锁模式：volatile + synchronized

**使用**：

getSingleton方法获取的就是单例对象，保证每次获取的都是一样的bean！hash值什么的完全一样！

### 适配器模式

> 提供一种拟合能力，可以让本不能调用某一方法或不适合的类能够调用该方法

**使用**：

销毁方法多种，是AbstractApplicationContext注册虚拟机之后，虚拟机关闭之前执行的动作。在销毁方法执行时不关注底层的具体实现，通过统一接口进行销毁，于是使用适配器`DisposableBeanAdaptor`，实现统一的处理，让其他类可以轻松调用

### 观察者模式

> 解耦，一个对象状态发生改变时候，所有依赖于它的对象都得到通知并被自动更新

**使用**：

在Event事件功能中提供的事件定义、发布、监听事件完成自定义动作！可以掌握自己的信息

### 策略模式

> 定义一系列的算法，把它们一个个封装起来，并且使它们可相互替换

**使用**：

定义实例化策略，可以动态的选用cglib或jdk进行实例化！一般来说是接口默认选择jdk，反之会选择cglib。但可以强制使用jdk/cglib

### 装饰者模式

> 动态地将责任附加到对象上，而不用去修改类的代码

**使用**：

以BeanFactory接口为例，具体的功能是通过层层装饰来完成的，在一次次的功能扩展后最终才会得到DefaultListableBeanFactory！

## 相关技术

### Hook/钩子

> 一种消息拦截机制，并能够对拦截信息进行自动处理
>
> 1. 线程钩子：可以拦截单个进程的消息
> 2. 系统钩子：能够拦截全部进程的消息

使用方法(使用场景在方法备注中)

```java
// 1.程序正常退出，最新的非守护线程退出或该退出方法被唤醒
// 2.JVM被中断
Runtime.getRuntime().addShutdownHook(Thread）；
```

测试一下Hook在Java中的使用！

```java
@Slf4j
public class HookTest {
    private void selfIntroduction() {
        log.info(this.getClass().getSimpleName());
    }
    @Test
    public void testHook() {
        // 程序退出时候调用selfIntroduction
        Runtime.getRuntime().addShutdownHook(new Thread(this::selfIntroduction));
        log.info("do nothing");
    }
}
```

```txt
15:49:15.394 [main] INFO edu.neu.spring.test.HookTest - do nothing
15:49:15.394 [Thread-0] INFO edu.neu.spring.test.HookTest - HookTest
```

### HuTool

> Hutool = Hu + tool，是原公司项目底层代码剥离后的开源库，“Hu”是公司名称的表示，tool表示工具。Hutool谐音“糊涂”，一方面简洁易懂，一方面寓意“难得糊涂”。

**组成结构**：一个Java基础工具类，对文件、流、加密解密、转码、正则、线程、XML等JDK方法进行封装，组成各种Util工具类等等嘞！

### aopalliance

`aopalliance`属于AOP联盟，定义了一些标准

**org.aopalliance.aop包**：分为标记接口Advice和运行时异常AspectException

- Advice：通知的标记接口。实现可以是任意类型，比如下面的Interceptor
- AspectException：所有的AOP框架产生异常的父类。它是个RuntimeException

**org.aopalliance.intercept包**：主要分为两大接口，Interceptor、JoinPoint

- Interceptor：它继承自Advice，它通过拦截器得方式实现通知的效果(也属于标记接口)
- MethodInterceptor：具体的接口。拦截方法 （Spring提供了非常多的具体实现类）
- ConstructorInterceptor：具体接口。拦截构造器 （Spring并没有提供实现类）
- Joinpoint：AOP运行时的连接点（顶层接口）
- Invocation：继承自Joinpoint。 表示执行，提供了Object[] getArguments()来获取执行所需的参数
- MethodInvocation：（和MethodInterceptor对应，它的invoke方法入参就是它）表示一个和方法有关的执行器。提供方法Method getMethod() （Spring提供了唯一（唯二）实现类：ProxyMethodInvocation）
- ConstructorInvocation：和构造器有关。Constructor<?> getConstructor(); (Spring没有提供任何实现类)

# 待完成

## Cglib多重代理的实现

Cglib对字节码增强时，多重代理会使用多次newInstance，导致类加载错误

> 预计采用责任链模式进行实现

## 数据库支持

Spring提供了对JDBC的支持，项目中暂未实现

