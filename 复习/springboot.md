springboot

springboot启动流程

```java
    public static void main(String[] args) {
        SpringApplication.run(QuartzSApplication.class, args);
    }

/**
	 * Static helper that can be used to run a {@link SpringApplication} from the
	 * specified source using default settings.
	 * @param primarySource the primary source to load
	 * @param args the application arguments (usually passed from a Java main method)
	 * @return the running {@link ApplicationContext}
	 */
	public static ConfigurableApplicationContext run(Class<?> primarySource, String... args) {
		return run(new Class<?>[] { primarySource }, args);
	}

	/**
	 * Static helper that can be used to run a {@link SpringApplication} from the
	 * specified sources using default settings and user supplied arguments.
	 * @param primarySources the primary sources to load
	 * @param args the application arguments (usually passed from a Java main method)
	 * @return the running {@link ApplicationContext}
	 */
	public static ConfigurableApplicationContext run(Class<?>[] primarySources, String[] args) {
		return new SpringApplication(primarySources).run(args);
	}
```

上述代码表明springboot在启动时先初始化SpringApplication对象然后再执行这个对象的run方法，

分为步骤1和步骤2

步骤1：

```java
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
        //在这个代码的调用方法那里resourceLoader为null  这行代码没有意义
		this.resourceLoader = resourceLoader;
        //校验传入的QuartzSApplication.class的类对象是否为null
		Assert.notNull(primarySources, "PrimarySources must not be null");
	    //将传入的主配置类数组转换为 LinkedHashSet 集合，并赋值给 primarySources 属性。主配置类通常是 @SpringBootApplication 注解标注的类。	
    this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
	//根据类路径来推断应用的 Web 类型，可以是 SERVLET、REACTIVE 或 NONE。	
    this.webApplicationType = WebApplicationType.deduceFromClasspath();
	//	通过 getSpringFactoriesInstances 方法获取 BootstrapRegistryInitializer 接口的所有实现类，并将它们存储在 bootstrapRegistryInitializers 列表中。BootstrapRegistryInitializer 用于初始化 Spring 应用上下文的引导注册表。
    this.bootstrapRegistryInitializers = new ArrayList<>(
				getSpringFactoriesInstances(BootstrapRegistryInitializer.class));
	//通过 getSpringFactoriesInstances 方法获取 ApplicationContextInitializer 接口的所有实现类，并将它们设置为应用上下文的初始化器。	
    setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));
	//通过 getSpringFactoriesInstances 方法获取 ApplicationListener 接口的所有实现类，并将它们设置为应用上下文的事件监听器。	
    setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
    //推断应用的主类，通常是包含 main 方法的类。这个方法仅仅是找到main方法所在的类，为后面的扫包作准备
		this.mainApplicationClass = deduceMainApplicationClass();
	}
```

我们知道在spring boot项目中，只要用注解@Configuration、@Bean、@Compont等注解标注的类spring boot会自动为他们创建bean。同时被注解编注的类创建bean有一个前提，只对启动类所在的basepackage下的所有带有@Component等注解的类才会创建bean。（@ComponentScan默认只扫描同包、子包下的所有类）。spring boot 默认的包扫描范围 问题来了，如果是加入maven坐标依赖的jar包，就是项目根目录以外的Bean是怎么添加的？？如果你了解过spring boot自动装配的原理，那么你可以很容易知道，在项目根目录以外的Bean，也就是导入的spring-boot-starter-***的maven依赖 是根据 /META-INF/spring.factories下的文件去进行加载的。

Spring 应用上下文的引导注册表,应用上下文的初始化器,还有应用上下文的事件监听器都是从/META-INF/spring.factories下的文件去进行加载的

/META-INF/spring.factories自动装配的原理：

步骤2：

```java
public ConfigurableApplicationContext run(String... args) {
	//记录启动时间，用于计算应用启动所花费的时间。	
    long startTime = System.nanoTime();
	//创建引导上下文，用于应用启动过程中的一些基础操作。	管理初始化器（BootstrapInitializer）和监听器（BootstrapListener）等
    //bootstrapContext会在prepareContext中被使用事件的方式关掉
    DefaultBootstrapContext bootstrapContext = createBootstrapContext();
		
    ConfigurableApplicationContext context = null;
	//配置头信息属性，用于指示应用是否在headless模式下运行。	 啥是头信息属性，干嘛用的，啥是headless模式
    //headless模式:在服务器可能缺少显示设备、键盘、鼠标等外设的情况下 还可以正常运行
    configureHeadlessProperty();
    //获取应用程序的运行监听器，用于处理应用启动过程中的事件。
		SpringApplicationRunListeners listeners = getRunListeners(args);
	//通知监听器应用即将启动，传递引导上下文和主应用类信息。	
    listeners.starting(bootstrapContext, this.mainApplicationClass);
		try {
            //创建应用程序参数对象，用于保存命令行参数。
			ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
            //准备应用环境，包括配置加载、属性解析等操作。主要就是加载了一些配置，他会把默认配置放在最后，确保默认配置最后生效
			ConfigurableEnvironment environment = prepareEnvironment(listeners, bootstrapContext, applicationArguments);
			//配置忽略 Bean 信息，用于提高应用启动速度。
            configureIgnoreBeanInfo(environment);
			//打印启动横幅，显示应用的名称和版本信息。
            Banner printedBanner = printBanner(environment);
			//创建应用上下文，根据应用的类型创建不同类型的上下文，如 WebApplicationContext 或 GenericApplicationContext。
            context = createApplicationContext();
			//设置应用启动时的策略，用于记录应用启动过程中的耗时等信息。
            context.setApplicationStartup(this.applicationStartup);
			//准备应用上下文，包括注册 BeanDefinition、扫描组件等操作。
            prepareContext(bootstrapContext, context, environment, listeners, applicationArguments, printedBanner);
			//刷新应用上下文，启动 Spring 应用上下文，初始化所有 Bean。
            refreshContext(context);
			//在应用上下文刷新之后执行一些额外的操作。
            afterRefresh(context, applicationArguments);
			//计算应用启动所花费的时间。
            Duration timeTakenToStartup = Duration.ofNanos(System.nanoTime() - startTime);
            //如果配置了打印启动信息，则记录应用启动的相关信息。
			if (this.logStartupInfo) {
				new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), timeTakenToStartup);
			}
            //通知监听器应用已经启动完成，传递应用上下文和启动耗时信息。
			listeners.started(context, timeTakenToStartup);
			//调用应用中所有实现了 ApplicationRunner 和 CommandLineRunner 接口的 Bean 的 run 方法。
            callRunners(context, applicationArguments);
		}
		catch (Throwable ex) {
			handleRunFailure(context, ex, listeners);
			throw new IllegalStateException(ex);
		}
		try {
			Duration timeTakenToReady = Duration.ofNanos(System.nanoTime() - startTime);
            //通知监听器应用已经准备就绪，传递应用上下文和准备就绪耗时信息。
			listeners.ready(context, timeTakenToReady);
		}
		catch (Throwable ex) {
			handleRunFailure(context, ex, null);
			throw new IllegalStateException(ex);
		}
		return context;
	}
```

1.`configureIgnoreBeanInfo(environment)` 这行代码的作用是配置忽略 Bean 信息，主要是为了提高应用的启动速度。在 Spring 应用启动时，会使用 Java 的内省（Introspection）机制来分析 Bean 的属性和方法，以便进行依赖注入等操作。然而，这种内省机制可能会导致一些性能问题，特别是当应用中存在大量的 Bean 时。

通过配置忽略 Bean 信息，可以告诉 Spring 在启动时不使用内省机制来分析 Bean，而是直接使用预先定义的 Bean 信息，这样可以减少启动时的内省操作，从而提高应用的启动速度。不过需要注意的是，配置忽略 Bean 信息可能会导致某些特定的 Bean 无法被正确地初始化或注入依赖，因此需要谨慎使用。

2.  Introspection机制   在 Spring 启动时不使用内省机制来分析 Bean，而是直接使用预先定义的 Bean 信息，是通过配置 `configureIgnoreBeanInfo(environment)` 实现的。这里的预先定义的 Bean 信息通常是通过编程或配置文件指定的，而不是在运行时通过内省机制动态获取的。

   具体来说，当我们在 Spring 的配置文件（如 XML 配置文件、Java Config 类等）中定义 Bean 时，会提供 Bean 的类名、属性值等信息，Spring 在启动时会根据这些信息直接创建 Bean，而不需要使用内省机制来分析类的属性和方法。

   另外，Spring 还提供了一些特殊的注解（如 `@Component`、`@Service`、`@Repository`、`@Controller` 等）来标识类，表示这些类是 Spring 的 Bean，Spring 在启动时会自动扫描这些注解，并根据注解中的信息直接创建 Bean，也不需要使用内省机制来分析类的属性和方法。

   总之，通过预先定义 Bean 的信息，Spring 在启动时可以直接根据这些信息创建 Bean，而不需要使用内省机制来动态获取 Bean 的属性和方法。这样可以提高应用的启动速度，但需要开发人员在编写配置时确保提供正确的 Bean 信息。

3. refreshContext(context);这句代码中会开始进行ioc的自动扫描类并实例化对象存进ioc容器中，

   在看代码的过程中出现了DefaultSingletonBeanRegistry这个类，好像单例对象都是从这里拿的，这个类里有好多的concurrenthashmap   

   单例是由spring容器管理的，多例对象每次调用都会创建新对象一般由调用者管理

   ```java
   /** Maximum number of suppressed exceptions to preserve. */
   	private static final int SUPPRESSED_EXCEPTIONS_LIMIT = 100;
   
   //缓存单例对象：bean名字和bean实例
   //一级缓存：用于存放完全初始化好的bean
   	/** Cache of singleton objects: bean name to bean instance. */
   	private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);
   //缓存bean名字和对象工厂对象
   //三级缓存：存放bean工厂对象，用于解决循环依赖
   	/** Cache of singleton factories: bean name to ObjectFactory. */
   	private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);
   //缓存早期单例对象   bean名字和bean实例
   //二级缓存：存放原始的bean对象（尚未填充属性），用于解决循环依赖
   	/** Cache of early singleton objects: bean name to bean instance. */
   	private final Map<String, Object> earlySingletonObjects = new ConcurrentHashMap<>(16);
   //注册单机对象的集合，包含bean 名字按注册顺序
   	/** Set of registered singletons, containing the bean names in registration order. */
   	private final Set<String> registeredSingletons = new LinkedHashSet<>(256);
   //正在创建的bean的名字集合
   //这种创建方式会拿到concurrenthashmap的keySet
   	/** Names of beans that are currently in creation. */
   	private final Set<String> singletonsCurrentlyInCreation =
   			Collections.newSetFromMap(new ConcurrentHashMap<>(16));
   //当前从创建中检查中排除的bean的名字
   	/** Names of beans currently excluded from in creation checks. */
   	private final Set<String> inCreationCheckExclusions =
   			Collections.newSetFromMap(new ConcurrentHashMap<>(16));
   //异常的集合
   	/** Collection of suppressed Exceptions, available for associating related causes. */
   	@Nullable
   	private Set<Exception> suppressedExceptions;
   
   	/** Flag that indicates whether we're currently within destroySingletons. */
   	private boolean singletonsCurrentlyInDestruction = false;
   //需要执行清理工作的bean实例  bean的名字和需要清理的实例 ？
   	/** Disposable bean instances: bean name to disposable instance. */
   	private final Map<String, DisposableBean> disposableBeans = new LinkedHashMap<>();
   //bean名字和这个bean包含的bean名字集合
   	/** Map between containing bean names: bean name to Set of bean names that the bean contains. */
   	private final Map<String, Set<String>> containedBeanMap = new ConcurrentHashMap<>(16);
   //bean名字和依赖这个bean的名字集合
   	/** Map between dependent bean names: bean name to Set of dependent bean names. */
   	private final Map<String, Set<String>> dependentBeanMap = new ConcurrentHashMap<>(64);
   //bean名字和zhe'ge
   	/** Map between depending bean names: bean name to Set of bean names for the bean's dependencies. */
   	private final Map<String, Set<String>> dependenciesForBeanMap = new ConcurrentHashMap<>(64);
   ```

   三级缓存：

   先从一级获取，失败再从二级，三级获取

   - `singletonObject`：一级缓存，该缓存`key = beanName, value = bean;`这里的bean是已经创建完成的，该bean经历过**实例化->属性填充->初始化**以及各类的后置处理。因此，一旦需要获取bean时，我们第一时间就会寻找一级缓存
   - `earlySingletonObjects`：二级缓存，该缓存`key = beanName, value = bean;`这里跟一级缓存的区别在于，该缓存所获取到的bean是提前曝光出来的，是还没创建完成的。也就是说获取到的bean只能确保已经进行了实例化，但是属性填充跟初始化还没有做完(AOP情况后续分析)，因此该bean还没创建完成，仅仅能作为指针提前曝光，被其他bean所引用
   - `singletonFactories`：三级缓存，该缓存`key = beanName, value = beanFactory;`在`bean`实例化完之后，属性填充以及初始化之前，如果允许提前曝光，`spring`会将实例化后的`bean`提前曝光，也就是把该`bean`转换成`beanFactory`并加入到三级缓存。在需要引用提前曝光对象时再通过`singletonFactory.getObject()`获取。
   - `registeredSingletons`：保存着所有注册过`单例的beanName`，是一个`set`确保不重复

   检测循环依赖的过程如下：

   - A 创建过程中需要 B，于是 **A 将自己放到三级缓里面** ，去实例化 B
   - B 实例化的时候发现需要 A，于是 B 先查一级缓存，没有，再查二级缓存，还是没有，再查三级缓存，找到了！
   - - **然后把三级缓存里面的这个 A 放到二级缓存里面，并删除三级缓存里面的 A**
     - B 顺利初始化完毕，**将自己放到一级缓存里面**（此时B里面的A依然是创建中状态）
   - 然后回来接着创建 A，此时 B 已经创建结束，直接从一级缓存里面拿到 B ，然后完成创建，**并将自己放到一级缓存里面**
   - 如此一来便解决了循环依赖的问题

两层缓存就能解决循环依赖问题了，为什么还要用三层缓存

为什么要3层，不要2层，主要就是第三层的Bean对象工厂，这个Bean对象工厂用于创建返回这个Bean对象的单例，需要判断该对象是否有AOP，如果有AOP，那么返回Bean对象的代理对象，没有AOP，返回Bean对象本身。
