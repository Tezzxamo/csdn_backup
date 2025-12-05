@[toc]
## 概要
`springboot版本：2.7.11（不同版本间仅存在细微差距，大的流程是相似的）`
`仅深入部分源码解析，refresh()方法源码有官方注释，不再此处单独列出解析`

**Springboot的启动流程可以看作是一个有序的、阶段化的过程，其核心是`初始化IOC容器、并启动内嵌式Web服务器`，整个过程始于对main()中的run()的调用。**


```java
    public static void main(String[] args) {
        SpringApplication.run(ApplicationStarter.class, args);
    }
```

## 思维导图
`下图是两个步骤的图示步骤拆解，更详细步骤的看代码块中的注释`
![SpringBoot启动流程思维导图](../resources/pictures/sb_1.png)


## 步骤一：在main()中调用run()的时候，其内部会先创建一个`SpringApplication`实例
`在创建该实例的时候，主要做了以下几件事情：`

 1. 推断应用是`Web应用`还是`非Web应用`。
 2. 加载并初始化所有实现了`BootstrapRegistryInitializer`接口的实例（引导上下文初始化器）
 3. 设置初始化器（主应用程序上下文初始化器）
 4. 设置应用监听器
 5. 推断主配置类，即被`@SpringBootApplication`注解的类

 【其中2、3、4，都使用了`getSpringFactoriesInstances()`，该方法用于查找并实例化所有实现了特定接口的类的方法。】
 
`代码块1-1：`
```java
	public static ConfigurableApplicationContext run(Class<?> primarySource, String... args) {
		return run(new Class<?>[] { primarySource }, args);
	}
	public static ConfigurableApplicationContext run(Class<?>[] primarySources, String[] args) {
		return new SpringApplication(primarySources).run(args);
	}
```

`代码块1-2：`
```java
	public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
		this.resourceLoader = resourceLoader;
		Assert.notNull(primarySources, "PrimarySources must not be null");
		this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
		// 1、推断应用是 Web应用 还是 非Web应用 
		this.webApplicationType = WebApplicationType.deduceFromClasspath();
		// 2、加载并初始化所有实现了 BootstrapRegistryInitializer 接口的实例
		this.bootstrapRegistryInitializers = new ArrayList<>(
				getSpringFactoriesInstances(BootstrapRegistryInitializer.class));
		// 3、设置初始化器
		setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));
		// 4、设置监听器
		setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
		// 5、推断主配置类
		this.mainApplicationClass = deduceMainApplicationClass();
	}
```


## 步骤二：调用run()方法，走完流程后返回一个运行中的`ApplicationContext`：
`整个流程由 SpringApplicationRunListener  驱动，通过事件解耦，每个过程基本上都有扩展点`

详细步骤在代码块中，此处进行简单拆解为四个阶段：
 - 准备：创建引导上下文、声明主程序上下文、准备环境、加载监听器 
 - 配置：配置环境、配置主程序上下文（配置环境、配置bean工厂、应用初始化器、关键单例bean注册、懒加载配置、属性源排序、加载主配置源、上下文加载完成的事件发布）
 - 刷新：刷新主程序上下文，完成bean加载、自动装配、web服务器启动、AOP代理等
 - 完成：执行 Runner，发布就绪事件

`代码块2-1：`
```java
	public ConfigurableApplicationContext run(String... args) {
		// 记录启动时间，用于后续计算耗时
		long startTime = System.nanoTime();
		// 1、创建一个引导上下文，用于在 主ApplicationContext 创建之前提供早期服务，主要来源于步骤一的第二步骤
		DefaultBootstrapContext bootstrapContext = createBootstrapContext();
		// 声明 主应用上下文变量，稍后会被初始化
		ConfigurableApplicationContext context = null;
		// 设置系统属性 java.awt.headless = true(默认) ，避免一些问题
		configureHeadlessProperty();
		// 获取所有应用监听器
		SpringApplicationRunListeners listeners = getRunListeners(args);
		// 发布starting事件，表明现在应用程序刚刚开始启动
		listeners.starting(bootstrapContext, this.mainApplicationClass);
		try {
			// 将main()方法传入的args封装为ApplicationArguments对象
			ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
			// 2、准备并配置 environment（环境）【此处先跳转到下一个代码块2-2】
			ConfigurableEnvironment environment = prepareEnvironment(listeners, bootstrapContext, applicationArguments);
			// 配置spring.beaninfo.ignore，如果没有自定义该配置，则默认为true，主要是为了优化启动性能
			configureIgnoreBeanInfo(environment);
			// banner是启动时打印的文字，不影响启动和功能
			Banner printedBanner = printBanner(environment);
			// 3、创建 应用上下文实例，根据是否是web程序创建对应类型的上下文
			context = createApplicationContext();
			// 给应用上下文设置启动追踪器（默认为无操作的启动追踪器）
			context.setApplicationStartup(this.applicationStartup);
			// 3、创建后对上下文进行准备，将前面注册好的组件注入到上下文中【此处跳转到下下个代码块2-3中】
			prepareContext(bootstrapContext, context, environment, listeners, applicationArguments, printedBanner);
			// 4、刷新上下文：
			//  ①调用 BeanFactoryPostProcessor
			//  ②注册 BeanPostProcessor 
			//  ③完成bean的实例化、依赖注入、初始化（包括AOP代理生成）
			//  ④如果是 Web 应用，启动内嵌 Web 容器
			//  ⑤发布 ContextRefreshedEvent
			//  注：自动配置（@EnableAutoConfiguration）就在这一步生效！通过条件注解（@ConditionalOnClass 等）决定哪些自动配置类被加载。
			refreshContext(context);
			// 留给子类扩展的钩子方法
			afterRefresh(context, applicationArguments);
			// 计算耗时并打印
			Duration timeTakenToStartup = Duration.ofNanos(System.nanoTime() - startTime);
			if (this.logStartupInfo) {
				new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), timeTakenToStartup);
			}
			// 发布 started 程序启动事件
			listeners.started(context, timeTakenToStartup);
			// 执行所有 CommandLineRunner 和 ApplicationRunner 的实现，这是开发者常用的启动后初始化钩子
			callRunners(context, applicationArguments);
		}
		catch (Throwable ex) {
			handleRunFailure(context, ex, listeners);
			throw new IllegalStateException(ex);
		}
		try {
			// 发布 ready 应用就绪事件
			Duration timeTakenToReady = Duration.ofNanos(System.nanoTime() - startTime);
			listeners.ready(context, timeTakenToReady);
		}
		catch (Throwable ex) {
			handleRunFailure(context, ex, null);
			throw new IllegalStateException(ex);
		}
		// 返回应用程序上下文
		return context;
	}
```

`代码块2-2：`
```java
	private ConfigurableEnvironment prepareEnvironment(SpringApplicationRunListeners listeners,
			DefaultBootstrapContext bootstrapContext, ApplicationArguments applicationArguments) {
		// 获取或创建一个合适的 environment 实例
		ConfigurableEnvironment environment = getOrCreateEnvironment();
		// 将命令行参数绑定到环境上
		// 举例：java -jar app.jar --server.port=8081 会覆盖 application.yml 中的 server.port
		configureEnvironment(environment, applicationArguments.getSourceArgs());
		// 将 ConfigurationPropertySources 附加到 environment 上
		ConfigurationPropertySources.attach(environment);
		// 发布 environment-prepared 事件 
		listeners.environmentPrepared(bootstrapContext, environment);
		// 将默认属性移到最末尾，因为其优先级最低，只有没有其他配置时才会应用默认属性配置
		DefaultPropertiesPropertySource.moveToEnd(environment);
		// 禁止通过配置文件设置 spring.main.environment-prefix
		Assert.state(!environment.containsProperty("spring.main.environment-prefix"),
				"Environment prefix cannot be set via properties.");
		// 将 Environment 中的属性绑定到当前 SpringApplication 实例的字段上
		bindToSpringApplication(environment);
		if (!this.isCustomEnvironment) {
			// 如果不是自定义环境：有必要的情况下将environment转换为目标类型
			EnvironmentConverter environmentConverter = new EnvironmentConverter(getClassLoader());
			environment = environmentConverter.convertEnvironmentIfNecessary(environment, deduceEnvironmentClass());
		}
		// 因为上一步骤可能调换了整个environment实例，所以重新调用一次该方法
		ConfigurationPropertySources.attach(environment);
		return environment;
	}
```
`代码块2-3：`

```java
	private void prepareContext(DefaultBootstrapContext bootstrapContext, ConfigurableApplicationContext context,
			ConfigurableEnvironment environment, SpringApplicationRunListeners listeners,
			ApplicationArguments applicationArguments, Banner printedBanner) {
		// 给上下文配置环境
		context.setEnvironment(environment);
		// 留给子类的扩展方法，此时刚刚创建了上下文并设置了环境，还没有加载BeanDefinition和创建bean，可以重写该方法以完成上下文级别的定制
		postProcessApplicationContext(context);
		// 应用所有 ApplicationContextInitializer，这是在refresh前最后修改上下文的机会
		applyInitializers(context);
		// 发布 context-prepared 事件，此时上下文已经完全装配完毕，自定义监听器可以在此处接入（如注册额外的bean，修改BeanDefintion等），这是事件驱动扩展的重要环节
		listeners.contextPrepared(context);
		// 引导上下文关闭，并将其管理的组件转移到主上下文中
		bootstrapContext.close(context);
		// 启动信息日志
		if (this.logStartupInfo) {
			logStartupInfo(context.getParent() == null);
			logStartupProfileInfo(context);
		}
		// 获取上下文的bean工厂，用于后续注册单例bean
		ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();
		// 注册核心bean：springApplicationArguments（封装和管理应用程序启动参数的核心接口）和springBootBanner（用于展示横幅、文字的）到容器中
		beanFactory.registerSingleton("springApplicationArguments", applicationArguments);
		if (printedBanner != null) {
			beanFactory.registerSingleton("springBootBanner", printedBanner);
		}
		// 配置bean工厂行为：
		// 	1、设置循环依赖处理策略：是否允许通过三级缓存解决构造器循环依赖，默认为false，仅支持setter注入
		// 	2、设置BeanDefinition覆盖策略：是否允许同名BeanDefinition覆盖，默认为false，可以在配置文件中更改配置
		if (beanFactory instanceof AbstractAutowireCapableBeanFactory) {
			((AbstractAutowireCapableBeanFactory) beanFactory).setAllowCircularReferences(this.allowCircularReferences);
			if (beanFactory instanceof DefaultListableBeanFactory) {
				((DefaultListableBeanFactory) beanFactory)
					.setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
			}
		}
		// 懒加载配置，如果设置了spring.main.lazy-initialization=true，则添加懒加载后处理器
		// 懒加载：bean在第一次被注入/获取时才创建，而不是启动时全部实例化，用于优化启动速度
		if (this.lazyInitialization) {
			context.addBeanFactoryPostProcessor(new LazyInitializationBeanFactoryPostProcessor());
		}
		// 添加一个属性源排序后处理器，用于确保ConfigurationPropertySources在其他PropertySource之前被处理，保证 @ConfigurationProperties 绑定的正确性。
		context.addBeanFactoryPostProcessor(new PropertySourceOrderingBeanFactoryPostProcessor(context));
		// 加载主配置源，并加载这些源到上下文中
		Set<Object> sources = getAllSources();
		Assert.notEmpty(sources, "Sources must not be empty");
		// load()会：
		// 	1、解析 @Configuration 类，
		// 	2、扫描 @Component、@Service 等组件
		//  3、处理 @Import、@ImportResource 等导入
		//  4、生成所有 BeanDefinition 并注册到 BeanFactory
		load(context, sources.toArray(new Object[0]));
		// 发布context-loaded事件，此时上下文已经完全准备好，但是bean还没有实例化（还没有调用refresh()），这是最后的扩展点
		listeners.contextLoaded(context);
	}
```

