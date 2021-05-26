# AnnotationConfigApplicationContext

[TOC]

## AnnotationConfigApplicationContext创建流程

### 父类静态代码块、代码块、无参构造方法

#### AbstractApplicationContext


1. 静态代码块加载类类ContextClosedEvent（提前加载避免奇怪的类加载问题，不提前加载WebLogic中关闭应用会出现问题）

2. 无参构造方法调用getResourcePatternResolver()方法初始化resourcePatternResolver

   1. getResourcePatternResolver()方法：初始化new了路径匹配资源解析器[PathMatchingResourcePatternResolver](#PathMatchingResourcePatternResolver)


#### GenericApplicationContext

无参构造方法初始化了BenFactory，new了一个[DefaultListableBeanFactory](#DefaultListableBeanFactory)

### AnnotationConfigApplicationContext无参构造方法

1. 初始化Bean定义读取器AnnotatedBeanDefinitionReader

   1. 调用getOrCreateEnvironment方法，初始化了环境配置Environment：StandardEnvironment

   2. 初始化了Condition条件处理器ConditionEvaluator：ConditionContextImpl 

   3. 调用registerAnnotationConfigProcessors方法

      1. 注册了顺序比较器

         ###### AnnotationAwareOrderComparator

         先比PriorityOrdered，再比Ordered，最后比，@Order

      2. 初始化设置了一个new出来的ContextAnnotationAutowireCandidateResolver

      3. 注册了[ConfigurationClassPostProcessor](#ConfigurationClassPostProcessor)的Bean定义，这个类是容器扫描处理核心类，在下面的refresh方法的工厂后置处理器中执行了它的逻辑

      4. 注册了[AutowiredAnnotationBeanPostProcessor](#AutowiredAnnotationBeanPostProcessor)、[CommonAnnotationBeanPostProcessor](#AutowiredAnnotationBeanPostProcessor)Bean定义，通过注解注入属性的逻辑在这里

      5. 如果有PERSISTENCE_ANNOTATION_PROCESSOR_CLASS_NAME  JPA类就注入PersistenceAnnotationBeanPostProcessor Bean定义

      6. 注册了EventListenerMethodProcessor  Bean定义

      7. 注册了DefaultEventListenerFactory  Bean定义

2. 初始化Bean定义扫描器[ClassPathBeanDefinitionScanner](#ClassPathBeanDefinitionScanner)

## 主流程refresh()方法

> ###### refresh

在AbstractApplicationContext中

在synchronized同步代码块中

### prepareRefresh()方法

做一些context refresh的准备工作

1. 初始化环境参数，子类可以重写initPropertySources()方法对参数做一些处理
2. 做一些参数校验，没有某个参数则不启动容器，抛出MissingRequiredPropertiesException异常
3. 初始化早期监听者集合、初始化早期事件集合

### obtainFreshBeanFactory()方法获取BeanFactory

1. 调用子类的refreshBeanFactory()方法
   1. GenericApplicationContext是：用标志位boolean refreshed，判断是true就抛出异常，意思是，只能调用refresh一次，是false就设置为true
   2. 其他有的是销毁了当前的BeanFactory，重新new 一个，可以多次refresh
2. 设置beanFactory的setSerializationId

### prepareBeanFactory(beanFactory)

准备使用BeanFactory

1. 获取类加载器ClassLoader，默认获取到的是AppClassLoader（类加载器内容看JVM篇章）

2. 判断是否忽略SpEL表达式功能，参数spring.spel.ignore，设置SpEL解析器StandardBeanExpressionResolver

3. addPropertyEditorRegistrar添加属性编辑注册器，ResourceEditorRegistrar

4. addBeanPostProcessor添加Bean后置处理器ApplicationContextAwareProcessor，用于对

   1. EnvironmentAware
   2. EmbeddedValueResolverAware
   3. ResourceLoaderAware
   4. ApplicationEventPublisherAware
   5. MessageSourceAware
   6. ApplicationContextAware
   7. ApplicationStartupAware

   这些Aware进行注入处理

5. 调用[ignoreDependencyInterface](#ignoreDependencyInterface)方法，添加了以下类：

   1. EnvironmentAware
   2. ResourceLoaderAware
   3. ApplicationEventPublisherAware
   4. MessageSourceAware
   5. ApplicationContextAware
   6. ApplicationStartupAware

6. 调用registerResolvableDependency方法，如果有Bean的属性是这些类型需要注入，那么注入我们下面指定的实例，向DefaultListableBeanFactory的resolvableDependencies集合中添加了：

   1. BeanFactory
   2. ResourceLoader
   3. ApplicationEventPublisher
   4. ApplicationContext

   ###### registerResolvableDependency

7. 添加了Bean后置处理器ApplicationListenerDetector，这个类是用来收集ApplicationListener的

8. ###### 处理LoadTimeWeaverAware

   1. 判断系统不是graalvm、容器中有LOAD_TIME_WEAVER_BEAN_NAME
   2. 向BeanFactroy中添加BeanPostProcessor：LoadTimeWeaverAwareProcessor，这个Bean后置处理器会向实现了LoadTimeWeaverAware的接口注入loadTimeWeaver
   3. 给BeanFactroy设置临时类加载器：ContextTypeMatchClassLoader

9. 向beanFactory中注册单例：环境参数、系统参数、系统环境参数、启动状态信息

### postProcessBeanFactory(beanFactory)

调用子类实现的postProcessBeanFactory方法在Bean工厂实例化Bean之前对BeanFactory进行一些处理，
如SpringBootWeb会在这进行以下处理：

1. 添加一些BeanPostProcessor，如WebApplicationContextServletContextAwareProcessor，用来处理ServletContextAware的
2. ignoreDependencyInterface一些新的Aware，如：ServletContextAware
3. 对于Scopes的处理，如ServletWebServerApplicationContext支持Session和Request作用域的Bean，会在这里调用registerScope注册SCOPE_REQUEST和SCOPE_SESSION
4. 提前扫描一些包路径，并注册扫描到的Bean定义

### invokeBeanFactoryPostProcessors(beanFactory)

先执行[PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors方法](#PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors方法)

执行完后判断不是graalvm，没有TempClassLoader，容器中有LOAD_TIME_WEAVER_BEAN_NAME，就添加后置处理器LoadTimeWeaverAwareProcessor、设置临时类加载器ContextTypeMatchClassLoader（上面[处理LoadTimeWeaverAware](#处理LoadTimeWeaverAware)这一步也有类似的操作，这两块只会有一块能执行，第一块执行会设置临时类加载器第二块执行的条件是没有临时类加载器）

> #### PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors方法

1. 执行传过来的参数beanFactoryPostProcessors中的BeanDefinitionRegistryPostProcessor类型的postProcessBeanDefinitionRegistry方法，这里SpringBoot会执行两个类

   1. CachingMetadataReaderFactoryPostProcessor
      1. 生成SharedMetadataReaderFactoryBean的Bean定义并注册到容器中，这个类可以生成CachingMetadataReaderFactory
      2. 调用configureConfigurationClassPostProcessor方法对ConfigurationClassPostProcessor的Bean定义的propertyValues添加了metadataReaderFactory参数，RuntimeBeanReference（CachingMetadataReaderFactory，就是步骤1中的）。这步作用就是对ConfigurationClassPostProcessor的metadataReaderFactory属性进行设值
   2. ConfigurationWarningsPostProcessor，调用所有的Check检查
      1. 调用ComponentScanPackageCheck检查@ComponentScan配置是否有问题（不是空的，不能扫描org、org.springframework）
      2. 有问题就打warn日志

2. 从BeanFactory中获取所有实现了PriorityOrdered接口的BeanDefinitionRegistryPostProcessor，排序（[AnnotationAwareOrderComparator](#AnnotationAwareOrderComparator)）,然后执行postProcessBeanDefinitionRegistry方法

   这里会执行[ConfigurationClassPostProcessor](#ConfigurationClassPostProcessor)这个类的postProcessBeanDefinitionRegistry方法，Spring重要逻辑，进行了包扫描、注入Bean定义

3. 重复上一步，执行实现了Ordered接口的BeanDefinitionRegistryPostProcessor的postProcessBeanDefinitionRegistry方法（PriorityOrdered和Ordered是继承关系，但是这里有将执行过的存起来，可以判断是否执行过）

4. 循环执行还未执行的BeanDefinitionRegistryPostProcessor的postProcessBeanDefinitionRegistry方法，这里由于执行完一个可能会增加所以得循环执行到没有了为止

5. 调用invokeBeanFactoryPostProcessors方法执行BeanFactoryPostProcessor的postProcessBeanFactory方法，这里会先执行BeanDefinitionRegistryPostProcessor类型的（按照之前执行postProcessBeanDefinitionRegistry的顺序），再执行方法参数传过来的但不是BeanDefinitionRegistryPostProcessor类型的

   这里会执行[ConfigurationClassPostProcessor](#ConfigurationClassPostProcessor)的postProcessBeanFactory方法对@Bean的方法进行cglib的代理

6. 再从BeanFactory中获取所有的BeanFactoryPostProcessor，处理还未执行过postProcessBeanFactory方法的，分类、排序，再执行。这里会执行以下类：

   1. PropertySourcesPlaceholderConfigurer（SpringBoot会执行）
      1. 获取容器environment合并localProperties（localOverride属性控制localProperties是否覆盖environment）
      2. 调用processProperties方法进行对占位符${}、代替符: 处理，也会对BeanName中的占位符、和注解中的占位符进行处理
   2. EventListenerMethodProcessor
      1. 找出容器中所有的EventListenerFactory
      2. 排序，然后缓存起来
   3. PreserveErrorControllerTargetClassPostProcessor
      1. 找出容器中所有的ErrorController（BasicErrorController，页面错误处理）
      2. 设置属性preserveTargetClass为true（使用CGLIB代理）

7. beanFactory.clearMetadataCache()清除缓存

   1. 将mergedBeanDefinitions中已创建的（alreadyCreated集合中存在的）bean定义需要合并的标志位stale设置true，因为后处理器可能修改了原始元数据，例如替换值中的占位符操作
   2. 将mergedBeanDefinitionHolders集合清空
   3. 清空allBeanNamesByType、singletonBeanNamesByType两个集合



### registerBeanPostProcessors(beanFactory)

按顺序实例化并注册Bean创建拦截器，用来拦截Bean的创建

1. 从BeanFactory获取所有的BeanPostProcessor的name
2. addBeanPostProcessor BeanPostProcessorChecker，这个后置处理器没什么大用，判断Bean定义的Role，打印日志这个Bea还没被所有的Bean后置处理器处理过
3. 按照PriorityOrdered、Ordered分组、排序（排序用的[AnnotationAwareOrderComparator](#AnnotationAwareOrderComparator)）添加到BeanFactory中addBeanPostProcessor。这个阶段对调用getBean(ppName, BeanPostProcessor.class)方法获得实例化的BeanPostProcessor
4. addBeanPostProcessor添加了ApplicationListenerDetector，这个类用来处理监听逻辑，这个BeanPostProcessor在普通Bean初始化完成后会判断是否是ApplicationListener类型的单例Bean，是的话添加到自己的和多播器的监听者集合中去，在Bean销毁之前会从多播器的监听者集合中删除

### initMessageSource()

初始化message source 国际化组件
如果容器里有就用容器里的，如果容器里没有就给一个包装类（但是没作用）并注册到容器里

### initApplicationEventMulticaster()

初始化事件监听多播器
如果容器中有，就用容器中的，如果没有就new一个SimpleApplicationEventMulticaster并注册到容器中

### onRefresh()

由子类实现初始化一些特殊的Bean

SpringBootWeb的ServletWebServerApplicationContext在这里有以下处理：

1. 调用父类的onRefresh方法初始化了ThemeSource：默认ResourceBundleThemeSource，用来切换主题

2. 调用createWebServer()方法创建Server

   1. 从容器中获取ServletWebServerFactory

   2. 先调用getSelfInitializer方法获取ServletContextInitializer

      1. 设置了ServletContext
      2. registerApplicationScope注册了application类型的作用域
      3. 注册环境Bean registerEnvironmentBeans
         1. 注册SERVLET_CONTEXT_BEAN_NAME，ServletContext
         2. 注册CONTEXT_PARAMETERS_BEAN_NAME
         3. 注册CONTEXT_ATTRIBUTES_BEAN_NAME，这里包含Tomcat的一些属性
      4. 获取所有ServletContextInitializer调用onStartup方法，这里添加了一些SpringMVC的基础Filter

   3. 调用getWebServer获取Server

      1. new Connector设置Tomcat参数之类的
      2. 返回了TomcatWebServer

   4. 向容器中注册new WebServerGracefulShutdownLifecycle(this.webServer)、new WebServerStartStopLifecycle(this, this.webServer)，后面refresh的finishRefresh方法中会调用启动WebServer

      > ###### WebServerGracefulShutdownLifecycle&WebServerStartStopLifecycle

   5. 调用initPropertySources方法替换了一些参数

### registerListeners()

1. 找出监听器Bean并向多播器中注册
2. 向多播器中添加所有本地监听器
3. 向多播器中添加容器中的监听器
4. 如果早期事件集合不为空，多播器将早期事件推送出去

### finishBeanFactoryInitialization(beanFactory)

实例化所有剩余的（非lazy init）单例

1. 判断是否有包含conversionService有的话实例化取出来setConversionService
2. 如果没有嵌入值解析器，addEmbeddedValueResolver设置一个默认的
3. 实例化所有的LoadTimeWeaverAware
4. beanFactory.setTempClassLoader(null);将临时类加载器置空
5. 将配置冻结configurationFrozen置为true，获取所有Bean定义名称缓存到frozenBeanDefinitionNames集合中，不再允许修改了
6. 预初始化所有单例
   1. 获取所有的BeanNames，遍历处理每一个
   2. getMergedLocalBeanDefinition获取合并后的Bean定义，因为经过Bean定义后置处理器，可能bean定义会有所改变
   3. 如果不是抽象的且是单例且不是懒加载的Bean，先判断是不是工厂Bean 是的话加前缀getBean，否则直接调用getBean方法实例化
   4. 如果是FactoryBean那么判断是不是SmartFactoryBean，不是的话就不getBean，是SmartFactoryBean的话调用isEagerInit方法允许预先实例化就getBean实例化
   5. 遍历所有的BeanName，判断是不是SmartInitializingSingleton
   6. 是的话就执行afterSingletonsInstantiated方法

### finishRefresh()

1. 清除resourceCaches中的class缓存

2. 初始化LifecycleProcessor生命周期管理器，如果容器中有就用容器中的，如果没有就new默认的DefaultLifecycleProcessor并放入容器中，这里SpringBoot是提前放进去了一个DefaultLifecycleProcessor

3. 调用LifecycleProcessor的onRefresh方法，DefaultLifecycleProcessor的onRefresh方法会调用startBeans方法并running设置为true

   startBeans方法

   1. 获取所有的Lifecycle类型的Bean（SpringBoot会获取到[WebServerGracefulShutdownLifecycle&WebServerStartStopLifecycle](#WebServerGracefulShutdownLifecycle&WebServerStartStopLifecycle)）
   2. 处理生成LifecycleGroup集合
   3. 循环调用LifecycleGroup的start方法
      1. 调用到WebServerStartStopLifecycle的start方法，会启动webServer.start()，将running设置true，并且发布ServletWebServerInitializedEvent事件
      2. 调用WebServerGracefulShutdownLifecycle的start方法，将running设置true

4. 发布一个ContextRefreshedEvent事件

5. 如果有org.graalvm.nativeimage.imagecode和spring.liveBeansView.mbeanDomain变量就向LiveBeansView中添加this容器

### catch (BeansException ex) destroyBeans();

如果有异常 销毁Bean 关闭刷新状态 抛出异常

### finally resetCommonCaches

最后在finally中清除了一些缓存

1. 反射缓存ReflectionUtils-方法、字段缓存
2. 注解缓存AnnotationUtils-注解扫描、注解
3. 处理器ResolvableType缓存
4. 类加载器缓存

## ConfigurationClassPostProcessor包扫描、CGLIB代理增强

> ###### ConfigurationClassPostProcessor

是在初始化调用[AnnotationConfigApplicationContext无参构造方法](#AnnotationConfigApplicationContext无参构造方法)时候加进去的，MyBatis的扫描Mapper也是在这里扫描出所有的Mapper注册Bean定义的

### postProcessBeanDefinitionRegistry方法扫描

(在refresh()方法的[invokeBeanFactoryPostProcessors](#invokeBeanFactoryPostProcessors)方法中会执行这个方法)

扫描并处理所有该加载的Bean定义

先获取传过来的BeanDefinitionRegistry（DefaultListableBeanFactory）的id（HashCode）
判断registriesPostProcessed集合、factoriesPostProcessed集合中是否已经存在这个id，存在就抛出IllegalStateException异常

向registriesPostProcessed集合中添加了这个id

然后调用[processConfigBeanDefinitions](#processConfigBeanDefinitions)方法进行处理

#### processConfigBeanDefinitions扫描、注入Bean定义

> ##### processConfigBeanDefinitions

1. 获取容器中所有的Bean定义，先判断每个Bean定义是否存在CONFIGURATION_CLASS_ATTRIBUTE属性，如果存在，代表Bean定义已经作为配置类处理过了，无需再处理，否则调用checkConfigurationClassCandidate方法筛选出我们的配置类，找不到就return

   > ###### checkConfigurationClassCandidate

   1. Bean定义没有ClassName或者Bean定义是由方法生成则不处理

   2. 忽略Spring底层组件的类（BeanFactoryPostProcessor、BeanPostProcessor、AopInfrastructureBean、EventListenerFactory）

   3. 获取Bean定义的AnnotationMetadata

   4. 获取@Configuration注解，设置Bean定义的CONFIGURATION_CLASS_ATTRIBUTE属性

      1. 有@Configuration注解，并且proxyBeanMethods属性是true就设置为CONFIGURATION_CLASS_FULL

      2. 调用isConfigurationCandidate判断为true，就设置为CONFIGURATION_CLASS_LITE

         > ###### isConfigurationCandidate

         isConfigurationCandidate方法：有@Configuration注解，判断不是interface类，有没有Component、ComponentScan、Import、ImportResource注解，或者类的方法上有没有@Bean注解，就返回true

      3. 其他情况不处理

      CONFIGURATION_CLASS_ATTRIBUTE属性作用：

      1. 判断Bean是否被处理过了
      2. @Configuration的proxyBeanMethods属性，默认为true，如果@Bean注解的方法被调用，返回的是从BeanFactory中获取的Bean而不是方法生成的Bean，如果是false的话，每次都返回方法生成的对象而不是BeanFactory里获取的（所以加了Configuration注解而且proxyBeanMethods没手动设置为false，这个Bean会被代理）

   5. 解析@Order注解并赋值到Bean定义中setAttribute(ORDER_ATTRIBUTE,order)

2. 根据bean定义的ORDER_ATTRIBUTE属性排序

3. 判断是不是单例Bean注册器SingletonBeanRegistry，是的话再判断有么有调用过setBeanNameGenerator方法，没调用过就从容器里获取名为CONFIGURATION_BEAN_NAME_GENERATOR的BeanName生成器BeanNameGenerator，容器里有就用容器里的，没有的话就是类默认的（这里用的就是默认的）

4. 判断Environment没有初始化就new一个StandardEnvironment（上面已经初始化过了）

5. 初始化配置类解析器new一个 ConfigurationClassParser，构造方法把参数都设置了

6. do while循环处理，直到待解析的集合为空为止（因为解析A的时候可能会发现B也需要解析）

   1. 调用[ConfigurationClassParser.parse](#ConfigurationClassParser)方法解析所有待解析的类
   2. 调用[ConfigurationClassParser.validate](#ConfigurationClassParser)方法校验解析结果
   3. 获取所有configurationClasses配置类，并筛选去掉alreadyParsed已解析过的（就是本次parse方法解析出来的新的），得到configClasses
   4. 初始化ConfigurationClassBeanDefinitionReader（如已初始化过则跳过）
   5. 调用[ConfigurationClassBeanDefinitionReader](#ConfigurationClassBeanDefinitionReader)的[loadBeanDefinitions](#loadBeanDefinitions)方法加载本次扫描到的BeanDefinition
   6. 将configClasses放入alreadyParsed，代表这些Class都处理过了
   7. 判断当前Bean定义注册器中的Bean定义总数如果大于解析之前的数量，代表这次解析出新的Bean定义了需要再解析新增的那些Bean定义
   8. 筛选出新增的配置类继续循环解析，直到没有新增的为止

7. 将ImportRegistry单例注入到容器中，这个类在[ImportAwareBeanPostProcessor](#ImportAwareBeanPostProcessor)中有用到，对实现了ImportAware接口的Bean，会在这里注入AnnotationMetadata（但实际上获取到的是null，不会注入）

8. 清除缓存metadataReaderFactory.clearCache，重置了metadataReaderCache（SpringBoot这里不会执行，因为注入了metadataReaderFactory的类型不是默认的CachingMetadataReaderFactory）



### postProcessBeanFactory代理@Bean方法

获取BeanFactory的id，如果缓存factoriesPostProcessed中包含这个id就报错

将这个id添加到factoriesPostProcessed中，意思是这个BeanFactory已经处理过了

如果集合registriesPostProcessed中不包含这个id，就执行processConfigBeanDefinitions方法，其实在执行postProcessBeanDefinitionRegistry方法时执行过了，也加到这个集合中了

调用[enhanceConfigurationClasses](#enhanceConfigurationClasses)方法对@Bean方法进行代理

向容器中添加ImportAwareBeanPostProcessor，这是InstantiationAwareBeanPostProcessor类型的，在创建Bean的时候会调用

> ###### ImportAwareBeanPostProcessor

1. postProcessProperties方法，向EnhancedConfiguration类型的Bean中注入BeanFactory
2. postProcessBeforeInitialization方法，向ImportAware类型的Bean注入这个Bean的AnnotationMetadata，从容器中获取ImportRegistry，然后从ImportRegistry中获取这个Bean的AnnotationMetadata注入，在[processImports](#processImports)方法解析[@Import](#@Import)时有向importStack中注册对应Bean的AnnotationMetadata

#### enhanceConfigurationClasses代理@Bean方法

> ###### enhanceConfigurationClasses

这里要处理的就是有@Configuration的配置类但类里没有@Bean注解的方法

1. 获取BeanFactory中所有的BeanName，遍历处理

   1. 获取Metadata，若是AnnotatedBeanDefinition类型的，调用getMetadata获取AnnotationMetadata，getFactoryMethodMetadata方法获取MethodMetadata
   2. 如果CONFIGURATION_CLASS_ATTRIBUTE属性不为空或者MethodMetadata不为空，并且Bean定义是AbstractBeanDefinition类型的
      1. 设置Bean定义的Class，如果是CONFIGURATION_CLASS_LITE，AnnotationMetadata不是空的，且这个类里没有@Bean注解的方法，将Bean定义的beanClass属性（有可能是className，也有可能是Class，是name就根据name获取Class）设置为Class，没别的处理了
      2. 如果是CONFIGURATION_CLASS_FULL，不是AbstractBeanDefinition类型的Bean定义就报错，将这个Bean定义添加到要CGLIB增强处理的结果集中

2. 初始化一个ConfigurationClassEnhancer增强器

3. 遍历要进行CGLIB增强的Bean，设置一下PRESERVE_TARGET_CLASS_ATTRIBUTE preserveTargetClass属性true，意思是AOP中shouldProxyTargetClass方法调用的时候要代理它

4. 调用ConfigurationClassEnhancer的enhance方法获取增强后的子类Class，

   这个ConfigurationClassEnhancer，创建了目标类的子类

   1. 增加了接口EnhancedConfiguration（继承BeanFactoryAware）
   2. 设置类名规则：SpringNamingPolicy，“BySpringCGLIB”
   3. BeanFactoryAwareGeneratorStrategy生成策略，添加了字段$$beanFactory
   4. 设置了三个CallBack代理
      1. BeanMethodInterceptor
         1. isMatch条件：
            1. 不是Object.class
            2. 不能是BeanFactoryAware的setBeanFactory方法（方法名是setBeanFactory且参数只有一个且是BeanFactory，方法所在的类是BeanFactoryAware）
            3. 方法上有@Bean注解
         2. intercept代理逻辑
            1. 根据方法获取BeanName，先从缓存中获取，获取不到就根据@Bean注解和方法名解析，解析完了放缓存里
            2. 还是处理BeanName，如果方法上还有@Scope注解并且proxyMode不是ScopedProxyMode.NO，而且这个Bean正在创建，就前缀`scopedTarget.`
            3. 如果容器中有这个Bean并且也有前缀&（FactoryBean）的Bean也有，意思是这个方法返回的Bean是个FactoryBean，返回一个代理类，代理的getObject方法从容器中获取而不是调用FactoryBean的getObject方法获取Bean，ScopedProxyFactoryBean类型的作用域工厂不做处理
            4. 判断当前正在执行的方法是不是要代理的这个方法，如果是就调目标类（说明目标类已经被代理过了）的方法获取Bean返回
            5. 最后从容器中获取Bean返回
      2. BeanFactoryAwareMethodInterceptor
         1. isMatch条件：是BeanFactoryAware的setBeanFactory方法
         2. intercept代理逻辑
            1. 找到$$beanFactory字段，找不到就报错
            2. 给$$beanFactory设BeanFactory
            3. 如果父类（代理之前的类）是BeanFactoryAware类型的，执行父类的setBeanFactory方法



> ###### ConfigurationClassParser

## ConfigurationClassParser解析配置类

### 配置类parse解析方法

遍历传过来的所有BeanDefinitionHolder

判断类型调用解析单个配置类的parse方法，生成一个ConfigurationClass

调用[processConfigurationClass](#processConfigurationClass)方法解析以下注解：

@Component、

@PropertySource、@PropertySources

@ComponentScan、@ComponentScans

@Import（ImportSelect）、

@ImportResource、

@Bean

执行this.deferredImportSelectorHandler.process()方法

1. 判断deferredImportSelectors如果是空的则不处理
2. 初始化一个DeferredImportSelectorGroupingHandler
3. 对deferredImportSelectors进行排序
4. 遍历deferredImportSelectors调用[DeferredImportSelectorGroupingHandler.register](#DeferredImportSelectorGroupingHandler.register)方法
5. 调用[DeferredImportSelectorGroupingHandler.processGroupImports](#DeferredImportSelectorGroupingHandler.processGroupImports)方法处理加了@Import的DeferredImportSelector类型的Bean



> ###### processConfigurationClass

### processConfigurationClass解析方法

1. 调用conditionEvaluator.shouldSkip方法根据@Conditional注解判断是否需要跳过该类，需要跳过就直接返回不做任何处理

2. 判断配置类缓存configurationClasses中是否已存在该类，不存在继续往下走，存在的话，调用isImported方法

   1. 是，如果已存在的配置类也是由别的配置类导入就合并信息，返回
   2. 否，从缓存configurationClasses中移除，并且从knownSuperclasses中移除相同的元素（老得失效了，继续解析）

   > ###### isImported

   isImported方法：配置类是通过@Import注解导入或者由别的配置类的配置而注册的

3. 组装SourceClass

4. 循环调用[doProcessConfigurationClass](#doProcessConfigurationClass)方法，直到SourceClass为空停止，这里循环是为了处理父类

5. 向配置类集合configurationClasses中添加这次传过来解析的配置类

### doProcessConfigurationClass解析方法

> ###### doProcessConfigurationClass

1. 如果配置类上有@Component注解，调用[processMemberClasses](#processMemberClasses)方法，先处理MemberClass（成员类）

2. 解析@PropertySources、@PropertySource注解，调用processPropertySource方法将指定配置文件的值解析处理到Environment中

3. **解析@ComponentScan、@ComponentScans注解**

   1. 判断扫描路径不是空、调用conditionEvaluator.shouldSkip方法根据@Conditional注解判断不能跳过该类，否则不处理
   2. 循环处理所有该扫描的路径
      1. 调用[ComponentScanAnnotationParser.parse](#ComponentScanAnnotationParser)方法解析出路径内所有符合条件的Class并包装成Bean定义持有器
      2. 遍历扫描出来的Bean定义持有器
         1. 获取Bean定义的原始定义（没有的话返回 null， 链式调用该方法，最终可获取到由用户定义的 BeanDefinition）
         2. 获取不到就用扫描出来的Bean定义
         3. 调用[checkConfigurationClassCandidate](#checkConfigurationClassCandidate)方法判断是否是配置类
         4. 是配置类就调用parse方法解析，parse方法又递归调用了[processConfigurationClass](#processConfigurationClass)方法解析

4. 解析@Import注解

   1. 先递归获类上的@Import注解和所有注解的@Import注解（比如A上有自定义的@ImportTest注解，但是@ImportTest注解上有@Import注解）
   2. 调用[processImports](#processImports)方法解析

5. 解析@ImportResource注解

   1. 获取locations（value）参数、reader参数的Class
   2. 遍历所有的locations，调用environment的resolveRequiredPlaceholders方法处理location的占位符:${}"、替换符":"，处理reader参数获取指定的BeanDefinitionReader的Class，并调用ConfigurationClass的addImportedResource方法添加到importedResources集合中

6. 解析@Bean注解

   1. 调用retrieveBeanMethodMetadata方法获取所有加了@Bean注解的方法

      > ###### retrieveBeanMethodMetadata

      1. 获取类中所有的@Bean注解方法，如果没获取到或者只获取到一个就返回
      2. 获取类大于一个到@Bean注解的方法需要通过ASM处理顺序（JVM获取方法的顺序每次调用不一定相同）

   2. 调用ConfigurationClass.addBeanMethod方法将获取到的方法添加到beanMethods集合中

7. 调用processInterfaces方法

   > ###### processInterfaces

   处理配置类实现的接口中default方法上的@Bean注解，迭代配置类上所有的接口类，执行以下逻辑

   1. 调用[retrieveBeanMethodMetadata](#retrieveBeanMethodMetadata)方法获取所有加了@Bean的方法
   2. 迭代获取到的这些方法
      1. 如果是抽象方法不做处理
      2. 不是抽象方法，调用ConfigurationClass.addBeanMethod方法将获取到的方法添加到beanMethods集合中
   3. 递归调用[processInterfaces](#processInterfaces)方法处理这次接口上实现的所有接口

8. 处理配置类的父类（外面调用的时候是do while，返回null时候停止，这里返回父类就会接着处理父类）

   1. 如果配置类没有父类，返回null
   2. 有父类
      1. 如果父类是java开头的活着集合knownSuperclasses中已经存在，返回null
      2. 将父类添加到knownSuperclasses集合中，并返回父类



> ###### processMemberClasses

### processMemberClasses处理成员类方法

1. 获取所有成员类
   1. 如果目标是Class，调用getDeclaredClasses方法获取
   2. 对于ASM单独处理获取
2. 遍历每一个成员类，进行筛选，调用[isConfigurationCandidate](#isConfigurationCandidate)方法判断是不是配置类，如果不是就不处理
3. 将需要处理的成员类进行排序
4. 遍历需要处理的成员类
   1. 判断importStack栈中是否存在该类，存在就抛出异常（这个栈的作用就是防止无限循环，比如A有内部类a，先解析的a，而a上配置了@Impor(A.class)走到这里的时候就会报错）
   2. 将类放入importStack栈中
   3. 调用[processConfigurationClass](#processConfigurationClass)方法处理
   4. 将类弹出importStack栈



> ###### processImports

### processImports解析@Import注解方法

1. 第一次执行解析方法需要调用isChainedImportOnStack方法防止无限递归（后面递归执行的时候就不判断了）

   isChainedImportOnStack方法

   1. 判断配置类是否在importStack栈中，不在就不用处理
   2. 从栈中找到这个类的AnnotationMetadata，判断ClassName是否一样，找不到就不处理，找到了就报错

2. 将本次要处理的配置类push到importStack栈中

3. 迭代所有找到的@Import注解，判断类型，有不同处理

4. 对于实现了ImportSelector接口的

   1. 调用ParserStrategyUtils.instantiateClass方法实例化ImportSelector

      > ###### ParserStrategyUtils.instantiateClass

      1. 判断如果是接口就报错
      2. 从BeanFactory中获取类加载器
      3. 调用createInstance方法创建Bean
         1. 获取所有的构造方法
         2. 如果只有一个构造方法并且构造方法有参数，解析参数，这里解析参数只支持注入Environment、ResourceLoader、BeanFactory、ClassLoader，有其他的就报错
         3. 实例化Bean

   2. 调用ImportSelector的getExclusionFilter方法获取过滤器与方法参数的Predicate合并

   3. 如果是DeferredImportSelector类型的，调用DeferredImportSelectorHandler.handle方法

      > ###### DeferredImportSelectorHandler.handle

      1. 将DeferredImportSelector包装为DeferredImportSelectorHolder

      2. 判断deferredImportSelectors是不是null，不是null就直接add进去

      3. 是null，初始化一个DeferredImportSelectorGroupingHandler

      4. 调用register方法

         > ###### DeferredImportSelectorGroupingHandler.register

         注册这个包装类holder，调用DeferredImportSelectorGroupingHandler的getImportGroup方法获取组别，并添加到对应的组别缓存groupings中，然后缓存到将配置类（最初的配置类）configurationClasses中

      5. 调用processGroupImports方法

         > ###### DeferredImportSelectorGroupingHandler.processGroupImports

         迭代groupings中的DeferredImportSelectorGrouping，将每个group里的所有DeferredImportSelectorHolder拿出来，执行[processImports](#processImports)方法

   4. 不是DeferredImportSelector类型（DeferredImportSelector比普通的ImportSelector要晚解析我们配置的Bean，DeferredImportSelector是在解析完所有要解析的Bean之后再解析的）

      1. 调用selectImports方法获取所有类名
      2. 组装SourceClass
      3. 调用[processImports](#processImports)方法处理

5. 对于实现了[ImportBeanDefinitionRegistrar](#ImportBeanDefinitionRegistrar)接口的（SpringAOP MyBatis等很多框架是通过这种方式向Spring中注入一些Bean的）

   1. 调用[ParserStrategyUtils.instantiateClass](#ParserStrategyUtils.instantiateClass)方法实例化ImportBeanDefinitionRegistrar
   2. 调用BeanFactory的addImportBeanDefinitionRegistrar方法添加到配置类的importBeanDefinitionRegistrars集合中，后面会解析处理的

6. 对于只加了@Import注解的或者是通过ImportSelector导入递归调用到这一步的

   > ###### @Import

   1. 调用importStack.registerImport方法添加到importStack的imports集合中（importStack是ImportRegistry的实现类，在[ConfigurationClassPostProcessor](#ConfigurationClassPostProcessor)的[processConfigBeanDefinitions](#processConfigBeanDefinitions)方法最后把这个importStack注册单例到容器里了，在[ImportAwareBeanPostProcessor](#ImportAwareBeanPostProcessor)中有用来给ImportAware注入对应的Bean的AnnotationMetadata）
   2. 调用[processConfigurationClass](#processConfigurationClass)方法解析

7. 最后从importStack栈中弹出（弹出的就是本次解析的配置类）

### 配置类validate校验方法

迭代configurationClasses集合调用ConfigurationClass.validate方法

获取配置类上的@Configuration注解，如果获取到了并且proxyBeanMethods属性是true

判断this.metadata.isFinal()处理程序如果结束了就报错

遍历配置类中所有的BeanMethod

判断如果方法是静态的就不处理，如果不是静态方法，判断配置类是不是加了@Configuration注解，如果是，再判断是不是允许重写方法，不能重写就报错（@Configuration类中的实例@Bean方法必须可以被重写，由CGLIB增强一些功能）





## ConfigurationClassBeanDefinitionReader加载Bean定义

### loadBeanDefinitions加载Bean定义方法

初始化一个TrackedConditionEvaluator

遍历传过来的所有配置类ConfigurationClass调用loadBeanDefinitionsForConfigurationClass方法

1. 调用TrackedConditionEvaluator.shouldSkip判断是否应该跳过

   1. 从缓存skipped中获取，如果获取到了就返回获取到的boolean
   2. 获取不到，判断是不是由其他类导入[isImported](#isImported)
      1. 是，遍历导入这个配置类的父配置类，递归调用shouldSkip判断是否跳过，有一个不跳过就不跳过，否则就返回true
      2. 否，调用ConfigurationClassBeanDefinitionReader类的conditionEvaluator判断shouldSkip（由@Conditional决定结果）
   3. 将结果放入缓存skipped中

2. 如果判断需要跳过则从BeanDefinitionRegistry和ImportRegistry中移除并中断

3. 如果是由别的配置类导入[isImported](#isImported)则调用registerBeanDefinitionForImportedConfigurationClass方法处理本类

   1. 获取配置类的注解元信息，初始化AnnotatedGenericBeanDefinition的Bean定义
   2. 生成BeanName
      1. determineBeanNameFromAnnotation方法根据注解信息先从@Component、@ManagedBean、@Named的值获取，如果配置了多个但是名不一样会报错
      2. 注解获取不到就用ClassName
   3. 获取@Scope注解信息，如果proxyMode为空就设默认值
   4. 调用[processCommonDefinitionAnnotations](#processCommonDefinitionAnnotations)方法处理基础注解值
   5. 包装为BeanDefinitionHolder
   6. 调用[applyScopedProxyMode](#applyScopedProxyMode)方法处理代理
   7. 将处理好的Bean定义包装注册到容器中
   8. 给配置类设置好BeanName

4. 遍历所有的BeanMethod，调用loadBeanDefinitionsForBeanMethod方法处理（处理@Bean）

   1. 调用shouldSkip方法根据@Conditional注解判断是否应该跳过，如果需要跳过添加到skippedBeanMethods集合中并中断
   2. 如果skippedBeanMethods集合中存在要处理的BeanMethod，中断
   3. 获取@Bean注解信息，解析BeanName和AliaseName别名
   4. 将所有的别名注册到容器中
   5. 调用isOverriddenByExistingDefinition判断
      1. 如果存在现有Bean定义，并且现有Bean定义是配置类的Bean定义（在配置类中@Bean生成的Bean定义）
         1. 如果是同一个类的同名方法（重写、重载、包括父类的同名的@Bean方法），则保留原来的Bean定义
         2. 如果是不同类的同名方法，覆盖原来的Bean定义
      2. 如果现有的Bean定义是ScannedGenericBeanDefinition类型的，意思是包扫描扫描出来的，覆盖原来的Bean定义（Spring 4.2开始的）
      3. 如果当前Bean定义的Role，大于ROLE_APPLICATION（意思是当前Bean定义是大于用户级别的Bean）覆盖原来的Bean定义
      4. 判断如果DefaultListableBeanFactory不允许重写Bean定义，抛异常
      5. 其他情况默认不重写
   6. 如果不需要重写判断BeanName是不是与配置类的BeanName相同，相同则抛异常，不同则结束处理，return
   7. 初始化一个ConfigurationClassBeanDefinition，并根据是不是内部类啊设置属性setSource、setBeanClass、setBeanClassName、setUniqueFactoryMethodName、setFactoryBeanName、setUniqueFactoryMethodName、setResolvedFactoryMethod、setAutowireMode、setAttribute(SKIP_REQUIRED_CHECK_ATTRIBUTE,true)
   8. 调用[processCommonDefinitionAnnotations](#processCommonDefinitionAnnotations)方法处理基础注解
   9. 根据@Bean的属性设置以下内容
      1. 设置autowire如果是byName或者byType注入，设置setAutowireMode注入属性，默认Autowire.NO不注入
      2. 获取autowireCandidate属性，默认true，如果是false，该Bean不作为注入的候选Bean
      3. 处理initMethod、destroyMethod属性
   10. 处理@Scope注解，如果没有默认ScopedProxyMode.NO，如果有，就用给定的值
   11. 如果proxyMode不是ScopedProxyMode.NO，调用[createScopedProxy](#createScopedProxy)方法，处理代理，并重新包装ConfigurationClassBeanDefinition
   12. 调用[registerBeanDefinition](#registerBeanDefinition)方法将ConfigurationClassBeanDefinition注册到容器中

5. 调用loadBeanDefinitionsFromImportedResources方法处理@ImportResource注解指定的配置文件

   1. 遍历所有的ImportedResource

   2. 判断如果指定的BeanDefinitionReader是默认的BeanDefinitionReader接口则给默认实现类

      1. 如果配置文件是.groovy的用GroovyBeanDefinitionReader
      2. 如果shouldIgnoreXml：spring.xml.ignore参数是true（代表要忽略xml配置文件）就报错
      3. 其他情况用XmlBeanDefinitionReader

   3. 从解析器实例缓存集合readerInstanceCache中获取BeanDefinitionReader，获取不到就初始化

      1. 反射调用newInstance方法实例化
      2. 如果是AbstractBeanDefinitionReader类型的，设置resourceLoader、environment
      3. 将实例化好的reader放到readerInstanceCache缓存中

   4. 调用BeanDefinitionReader的loadBeanDefinitions方法

      例如：XmlBeanDefinitionReader会读取、解析xml配置文件并根据配置文件注册Bean定义

6. 调用loadBeanDefinitionsFromRegistrars方法处理所有的Bean定义注册器

   > ###### loadBeanDefinitionsFromRegistrars
   >
   > ###### ImportBeanDefinitionRegistrar

   遍历所有的ImportBeanDefinitionRegistrar调用registerBeanDefinitions方法

   这里会调用：

   1. [AspectJAutoProxyRegistrar](#AspectJAutoProxyRegistrar)的registerBeanDefinitions方法处理AOP的逻辑



> ###### applyScopedProxyMode

### applyScopedProxyMode设置代理方法

如果传入的Bean定义ScopedProxyMode为NO，不需要代理则直接中断，返回传过来的Bean

判断TARGET_CLASS是否是强制CGLIB代理，然后调用createScopedProxy方法

> ###### createScopedProxy

1. 生成了一个ScopedProxyFactoryBean的RootBeanDefinition Bean定义
2. 为原始bean名称创建作用域代理定义，在容器中“隐藏”被代理的bean
   1. setDecoratedDefinition设置代理Bean定义要代理的目标Bean定义
   2. setOriginatingBeanDefinition设置代理Bean定义的原始Bean定义
   3. setSource setRole 复制属性
3. 在代理Bean定义中添加了targetBeanName property属性
4. 如果proxyTargetClass是ture，setAttribute PRESERVE_TARGET_CLASS_ATTRIBUTE true，否则添加proxyTargetClass false的property属性
5. 从原始bean定义复制autowire设置，setAutowireCandidate，setPrimary，copyQualifiersFrom
6. 忽略目标bean定义，以支持作用域代理
   1. 被代理的Bean setAutowireCandidate(false)，注入Bean的时候，被代理的Bean不作为候选Bean注入
   2. 被代理的Bean setPrimary(false)，不首选使用这个Bean
7. 将被代理的Bean注册到容器中，不过BeanName被加了前缀TARGET_NAME_PREFIX（scopedTarget.）
8. 返回一个设置好代理的Bean定义持有器



> ###### processCommonDefinitionAnnotations

### processCommonDefinitionAnnotations处理基础注解值

处理@Lazy、@Primary、@DependsOn、@Role、@Description的值

对AnnotatedBeanDefinition设置：

setLazyInit @Lazy

setPrimary @Primary

setDependsOn @DependsOn

setRole @Role

setDescription @Description



> ###### ComponentScanAnnotationParser

## ComponentScanAnnotationParser包扫描解析

### parse扫描解析方法

1. 初始化new一个[ClassPathBeanDefinitionScanner](#ClassPathBeanDefinitionScanner)，useDefaultFilters的值使用了@ComponentScan注解的属性
2. 给ClassPathBeanDefinitionScanner设置BeanNameGenerator，如果@ComponentScan注解的nameGenerator指定了BeanNameGenerator实现类使用指定的BeanName生成器，默认使用AnnotationBeanNameGenerator
3. 获取@ComponentScan的scopedProxy属性
   1. 如果指定了ScopedProxyMode，ClassPathBeanDefinitionScanner设置setScopedProxyMode指定的scopedProxy属性（设置scopeMetadataResolver属性new AnnotationScopeMetadataResolver(scopedProxy)）
   2. 如果没指定，获取@Component的scopeResolver属性，调用setScopeMetadataResolver方法设置Bean作用域解析器，默认使用AnnotationScopeMetadataResolver
4. 获取@ComponentScan的resourcePattern资源匹配后缀属性并设置，默认为`**/*.class`
5. 添加@ComponentScan注解上所有的includeFilters和excludeFilters
6. 根据@ComponentScan设置lazyInit属性，给Scanner的BeanDefinitionDefaults设置
7. 获取@ComponentScan注解上的basePackages属性，遍历处理
   1. 根据分隔符（`,; \t\n`逗号、分号、空格、制表符TAB、换行）分割，将分割结果添加到要扫描的包集set中
   2. 处理basePackageClasses，将所有指定Class所在到的包名添加到要扫描的包集set中
   3. 如果扫描的包集合是空的就将声明类declaringClass（方法参数，这里传入的是配置类名）的包路径添加进去
   4. 添加了一个排除过滤器，条件是className与declaringClass相同就排除掉
8. 调用ClassPathBeanDefinitionScanner的doScan方法扫描出Bean定义集合并返回



> ###### ClassPathBeanDefinitionScanner

## ClassPathBeanDefinitionScanner包扫描器

### 构造方法

1. 如果useDefaultFilters是ture调用registerDefaultFilters方法注册默认扫描过滤器
   1. 集合includeFilters添加了扫描@Component注解过滤器
   2. 获取JSR-250的javax.annotation.ManagedBean，没有就异常直接catch掉，有就加到includeFilters中
   3. JSR-330的javax.inject.Named，没有就异常直接catch掉，有就加到includeFilters中
2. setEnvironment设置了环境参数Environment
3. setResourceLoader方法
   1. 设置了一个new CachingMetadataReaderFactory
   2. 设置CandidateComponentsIndex的时候
      1. 参数spring.index.ignore不是ture（忽略@Indexed），@Indexed注解的Bean在编译时会加到META-INF/spring.components文件中，Spring启动的时候扫描包改为扫描这个文件以提升扫描速度
      2. 从COMPONENTS_RESOURCE_LOCATION **META-INF/spring.components**文件中获取需要加载的Bean（加了spring indexer依赖（annotationProcessor）在编译的时候通过Processor去生成这个文件，加载Bean的时候不再挨个扫描而是直接根据文件取加载Bean，以提升启动速度）
         1. 获取到了，会加载到一个new的CandidateComponentsIndex里，后面会根据这些配置加载一些Bean，后面调用findCandidateComponents方法的时候执行addCandidateComponentsFromIndex，直接取这个文件里列好的Bean去加载
         2. 获取不到就设置为null了，后面调用findCandidateComponents方法的时候是执行scanCandidateComponents自己扫描解析

### doScan扫描Class生成Bean定义

1. 断言参数不为空，初始化一个LinkedHashSet结果集
2. 遍历所有要扫描的包名
3. 调用父类ClassPathScanningCandidateComponentProvider的[findCandidateComponents](#findCandidateComponents)方法获取扫描出的Bean定义集合，然后遍历处理
   1. 调用resolveScopeMetadata方法处理@Scope注解设置ScopedProxyMode
   2. 用BeanName生成器获取BeanName
   3. 如果是AbstractBeanDefinition类型的Bean定义调用postProcessBeanDefinition方法
      1. applyDefaults给Bean定义设置BeanDefinitionDefaults的默认值
      2. 如果设置了Bean候选匹配规则autowireCandidatePatterns，根据规则设置这个Bean的autowireCandidate属性（注入该Bean时是否将这个Bean放入候选名单）
      3. 调用[processCommonDefinitionAnnotations](#processCommonDefinitionAnnotations)方法处理基础注解
      4. 调用checkCandidate方法检查容器中是否有同名的Bean定义
         1. 没有，继续处理
         2. 有且不是ScannedGenericBeanDefinition类型的Bean定义，或者是同类型（Source）的，说明包扫描重复了，不做处理，不再注册这个Bean定义了
         3. 有，是ScannedGenericBeanDefinition类型的，Source又不一样，同名了，报错
      5. 包装一个BeanDefinitionHolder
      6. 调用[applyScopedProxyMode](#applyScopedProxyMode)方法处理代理
      7. 将处理好的Bean定义添加到返回结果集中，并注册到容器中[registerBeanDefinition](#registerBeanDefinition)，有别名的话也注册一下别名



> ###### findCandidateComponents

### findCandidateComponents扫描指定目录的class并生成Bean定义

判断属性componentsIndex不为空（初始化的时候从META-INF/spring.components获取内容，获取不到就不会设置这个属性）并且所有includeFilter过滤器匹配的注解或者类上有@Indexed注解（编译的时候只会对加了@Indexed注解的相关类写入文件，比如我要扫描InterfaceA类型的类，但A上没有@Indexed注解，那么编译的时候就不会写入文件，扫描的时候从文件找不到就把这些类漏了）则调用[addCandidateComponentsFromIndex](#addCandidateComponentsFromIndex)方法从spring.components配置中加载Bean定义

否则调用[scanCandidateComponents](#scanCandidateComponents)方法从文件中扫描加载Bean定义

> ###### addCandidateComponentsFromIndex

### addCandidateComponentsFromIndex从spring.components配置中加载Bean

1. 初始化Bean定义set集合、初始化要处理的类名set集合
2. 遍历includeFilters
   1. 先获取TypeFilter支持的注解名或者类名，获取不到就报错（只支持AnnotationTypeFilter、AssignableTypeFilter这两个类型过滤器）
   2. 调用CandidateComponentsIndex的getCandidateTypes方法
      1. 从处理好的配置文件缓存index（key加了@Indexed的类（注解、接口、普通类），value符合条件的类（加了注解、实现了接口、继承了类），@Component注解上也有@Indexed的）中获取符合条件的类
      2. 挨个匹配扫描包路径，返回符合扫描包路径规则的类集合
   3. 添加到要处理的类名set集合中
3. 遍历要处理的类名集合
   1. 根据类名生成一个MetadataReader
   2. 判断这个类是否符合候选条件，先根据excludeFilters过滤，然后根据includeFilters过滤，最后调用ConditionEvaluator的shouldSkip方法根据@Conditional注解决定是否应该跳过，不符合条件就不做任何处理
   3. 初始化一个ScannedGenericBeanDefinition的Bean定义
   4. 设置setSource
   5. 再判断这个类是，独立的、不是接口或者是抽象类但有@Lookup注解（可以被代理），放入Bean定义集合中，否则不处理
   6. 返回选好的Bean定义集合

> ###### scanCandidateComponents

### scanCandidateComponents从指定目录加载Bean定义

1. 初始化一个Bean定义的set集合

2. 处理要扫描的目录String，处理占位符`$`和默认替换符号`:`从环境中取值设置上，将`.`换成`/`，前缀加上`classpath*:`，后缀加上`/**/*.class`

3. 调用PathMatchingResourcePatternResolver的getResources方法获取所有符合条件的class资源

   > ##### PathMatchingResourcePatternResolver

   1. 先分析要扫描的路径是不是正则匹配类型的，不是就简单处理
   2. 先找正则之前目录资源
   3. 再根据目录是什么类型的调不同的读取方法（虚拟磁盘vfs、vfszip，jar、war、zip、wsjar，普通的file目录）获取目录文件
   4. 然后一层一层遍历递归，判断是否符合正则匹配，筛选出结果。

4. 遍历获取到的资源

   1. 根据class资源生成一个MetadataReader
   2. 判断这个类是否符合候选条件，先根据excludeFilters过滤，然后根据includeFilters过滤，最后调用ConditionEvaluator的shouldSkip方法根据@Conditional注解决定是否应该跳过，不符合条件就不做任何处理
   3. 初始化一个ScannedGenericBeanDefinition的Bean定义
   4. 设置setSource
   5. 再判断这个类是，独立的、不是接口或者是抽象类但有@Lookup注解（可以被代理），放入Bean定义集合中，否则不处理
   6. 返回选好的Bean定义集合



# DefaultListableBeanFactory

## AbstractAutowireCapableBeanFactory无参构造方法

1. 调用了ignoreDependencyInterface方法向AbstractAutowireCapableBeanFactory的ignoredDependencyInterfaces集合中添加了BeanNameAware、BeanFactoryAware、BeanClassLoaderAware类。

   > ##### ignoreDependencyInterface
   >
   > ##### ignoreDependencyType

   两个方法的作用相似，忽略给定接口/类型的类自动装配功能，但不能忽略注解装配

2. 调用inNativeImage判断有没有参数org.graalvm.nativeimage.imagecode（是不是graalvm），初始化了 CglibSubclassingInstantiationStrategy



> ###### registerBeanDefinition

## registerBeanDefinition注册Bean定义

1. 校验BeanName、Bean定义参数不能为空

2. 如果是AbstractBeanDefinition类型的Bean定义调用AbstractBeanDefinition的validate验证方法

   1. 如果有要重写的方法但又存在工厂方法名，抛异常（无法将工厂方法与容器生成的方法重写相结合：工厂方法必须创建具体的bean实例）
   2. 校验如果有要重写的方法，校验方法是否存在，并将方法被重写完的属性设为false

3. 从Bean定义缓存beanDefinitionMap中获取

4. 如果获取到了

   1. 判断BeanFactory是否允许覆写Bean定义，不允许就抛异常

   2. 判断已存在的Bean定义Role等级小于要写入的Role登记，小于则打info日志

   3. equals判断两个Bean定义是否相同，不同则打debug日志

   4. 其他情况，打trace日志

      将Bean定义存入beanDefinitionMap缓存中

5. 如果获取不到，判断alreadyCreated集合是否为空

   1. 如果不为空，synchronized代码块

      1. 将Bean定义存入beanDefinitionMap缓存中

      2. 初始化一个ArrayList，将当前beanDefinitionNames集合加进去，并存入当前处理的BeanName

      3. beanDefinitionNames改为刚初始化的ArrayList

      4. 调用removeManualSingletonName方法

         > ###### removeManualSingletonName

         更新Bean工厂内部手动添加的单例名称集合，如果包含这个BeanName，从集合manualSingletonNames中remove掉（调用registerSingleton方法的时候会往这个集合中添加BeanName，并且添加到单例缓存中，意思是这是手动添加的单例），这里是自动注册了Bean定义，所以要从这个手动注册Bean定义的Name缓存中remove掉

   2. 如果是空的

      1. 将Bean定义存入beanDefinitionMap缓存中
      2. beanDefinitionNames添加BeanName
      3. 调用[removeManualSingletonName](#removeManualSingletonName)方法

   3. 将frozenBeanDefinitionNames集合设置为null

6. 如果Bean定义已存在，或者单例缓存中已存在这个BeanName，调用resetBeanDefinition方法重置Bean定义

   1. 调用clearMergedBeanDefinition方法将mergedBeanDefinitions集合中的Bean定义的stale属性设置为true，意思是需要重写合并这个Bean定义，从mergedBeanDefinitionHolders集合中移除掉这个BeanName
   2. 调用[destroySingleton](#destroySingleton)方法销毁这个Bean单例
   3. 从mergedDefinition集合中获取所有的MergedBeanDefinitionPostProcessor，执行resetBeanDefinition方法
   4. 遍历bean定义缓存中所有的Bean定义，判断如果Bean定义的父类是这个Bean，递归对这个子Bean定义执行resetBeanDefinition方法

7. 如果Bean定义不存在并且单例缓存中也没有并且configurationFrozen标志（是否可以为所有bean缓存bean定义元数据）还是true，调用[clearByTypeCache](#clearByTypeCache)方法清除根据类型找所有Name的缓存



> ###### destroySingleton

## destroySingleton销毁单例方法

先调用父类的方法

1. 从singletonObjects、singletonFactories、earlySingletonObjects、registeredSingletons缓存中移除这个Bean
2. 从disposableBeans集合（创建bean时候如果指定了销毁方法会添加到这个集合中）中获取DisposableBean
3. 调用destroyBean方法（假设要销毁BeanB，BeanC依赖BeanB，BeanB依赖BeanA，C-B-A）
   1. 从依赖缓存[dependentBeanMap](#dependentBeanMap)中移除并获取依赖BeanB的集合（获取到C）
   2. 迭代依赖这个Bean的所有Bean递归调用destroySingleton方法销毁（销毁C，因为C依赖B，要销毁B，C也就不完整了）
   3. 如果指定了销毁方法调用DisposableBean的destroy方法
   4. 从缓存[containedBeanMap](#containedBeanMap)集合中获取依赖BeanB的集合
   5. 迭代依赖BeanB的所有Bean递归调用destroySingleton方法销毁（跟上面的C类似）
   6. 在同步代码块中迭代dependentBeanMap的value，从set中移除这个beanName（从依赖BeanA的集合中移除BeanB）
   7. 从集合dependenciesForBeanMap中移除这个Bean

父类的销毁方法处理完后执行以下方法

1. 调用[removeManualSingletonName](#removeManualSingletonName)方法，从manualSingletonNames手动注册的BeanName集合中移除这个Bean

2. 调用clearByTypeCache方法

   > ###### clearByTypeCache

   清空通过Type寻找BeanName的缓存allBeanNamesByType、singletonBeanNamesByType

# Spring AOP的实现

> ###### AspectJAutoProxyRegistrar

## AspectJAutoProxyRegistrar注册代理处理器

这是一个[ImportBeanDefinitionRegistrar](#ImportBeanDefinitionRegistrar)接口的实现类

在refresh方法中的[invokeBeanFactoryPostProcessors](#invokeBeanFactoryPostProcessors)方法中[processImports](#processImports)解析@Import注解时实例化好并存到配置类的属性中

在[loadBeanDefinitions](#loadBeanDefinitions)的[loadBeanDefinitionsFromRegistrars](#loadBeanDefinitionsFromRegistrars)方法中被调用它的[registerBeanDefinitions](#AspectJAutoProxyRegistrar.registerBeanDefinitions)方法

> ### AspectJAutoProxyRegistrar.registerBeanDefinitions

1. 调用registerAspectJAnnotationAutoProxyCreatorIfNecessary方法注册AnnotationAwareAspectJAutoProxyCreator
   1. 如果容器中存在AUTO_PROXY_CREATOR_BEAN_NAME
      1. 判断容器中的是不是AnnotationAwareAspectJAutoProxyCreator类型的，是的话不做处理return
      2. 如果不是AnnotationAwareAspectJAutoProxyCreator类型，按照InfrastructureAdvisorAutoProxyCreator、AspectJAwareAdvisorAutoProxyCreator、AnnotationAwareAspectJAutoProxyCreator的顺序，优先使用排在后面的（如果不在这几个里面就是排在最后），如果容器中的AutoProxyCreator是前两个的类型就替换为AnnotationAwareAspectJAutoProxyCreator，然后return
   2. 如果容器中不存在AUTO_PROXY_CREATOR_BEAN_NAME
      1. 初始化一个AnnotationAwareAspectJAutoProxyCreator的RootBeanDefinition
      2. setSource设置source(这里是null的)
      3. 设置PropertyValue order HIGHEST_PRECEDENCE，就是顺序是最优先的
      4. setRole ROLE_INFRASTRUCTURE，最高级的Bean角色，后台角色
      5. 然后调用[registerBeanDefinition](#registerBeanDefinition)注册到容器中
2. 解析@EnableAspectJAutoProxy注解设置属性
   1. 如果proxyTargetClass属性是ture，把AnnotationAwareAspectJAutoProxyCreator的RootBeanDefinition设置PropertyValue proxyTargetClass true意思是强制使用CGLIB代理
   2. 如果exposeProxy属性是ture，把AnnotationAwareAspectJAutoProxyCreator的RootBeanDefinition设置PropertyValue exposeProxy true 意思是允许在ThreadLocal中缓存代理Bean



## AnnotationAwareAspectJAutoProxyCreator

