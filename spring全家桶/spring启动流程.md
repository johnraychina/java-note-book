


## 核心类
SpringApplication
ApplicationContext 
--ConfigurableApplicationContext
	--AnnotationConfigEmbeddedWebApplicationContext web环境
	--AnnotationConfigApplicationContext 普通环境
	
BeanWrapperImpl 设置属性，注入
DefaultListableBeanFactory


# Bean依赖注入

## 基于构造方法的 依赖注入
## 基于setter方法的 依赖注入
## Bean依赖解析过程
https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#beans-dependency-resolution

Spring容器按如下步骤解析bean依赖：
- 首先创建ApplicationContext, 并按元数据配置（描述bean）初始化。元数据配置可以用xml，java代码，注解方式来指定。
- 对每个bean，他依赖的东西，以属性、构造方法的参数、静态工厂方法的形式体现。当bean真正被创建的时候，这些依赖就会被提供给他。
- 每个属性或者构造方法参数，会设置为确切的值 或是指向容器的其他引用。
- 每个属性或者构造方法参数，会按实际需要转换为对应类型：默认情况下spring可以将string格式的配置转换为内置类型，比如int, long, String, boolean等.

Spring在创建容器后，就会校验bean的配置，但是只会在实际创建bean的时候设置其属性.
Singleton单例模式的bean默认在容器创建后，会被预先初始化。其他模式只会在要使用的时候初始化。
Bean的创建 对应着bean依赖图的创建。
解析依赖时产生的问题会在创建bean的时候显现出来。


## 循环依赖
如果你主要使用构造方法注入的方式，那很可能会产生spring无法自动解决的循环依赖。
A->B, B->A 会抛出异常 BeanCurrentlyInCreationException

一个解决办法是用setter注入替代构造方法注入（不推荐），官方推荐是修改代码结构，避免循环依赖。
和正常情况不同的是，循环依赖会强制一方先完全初始化。

通常你可以信任spring会做正确的事：他会在容器加载时检测配置问题（例如引用不存在的bean，或者循环依赖等）
Spring在创建bean时会尽可能晚设置属性和解析依赖，这意味着正确加载的Spring容器会在运行时产生异常（比如缺失属性），导致配置问题发现太晚。
为了解决这个问题，ApplicationContext创建后，默认情况下会（以启动时间和内存为代价）会预先初始化Singlton bean，当然你也可以把bean设置为懒加载.

如果没有循环依赖，spring容器会按bean的依赖顺序，先把底层bean配置好再注入给依赖他们的上层bean。
这意味着底层的bean会被先初始化、依赖的东西被注入、相关生命周期方法被调用(init方法).

org.springframework.beans.factory.BeanCurrentlyInCreationException: Error creating bean with name 'a': Requested bean is currently in creation: Is there an unresolvable circular reference?
	at org.springframework.beans.factory.support.DefaultSingletonBeanRegistry.beforeSingletonCreation(DefaultSingletonBeanRegistry.java:347) ~[spring-beans-5.2.7.RELEASE.jar:5.2.7.RELEASE]
	at org.springframework.beans.factory.support.DefaultSingletonBeanRegistry.getSingleton(DefaultSingletonBeanRegistry.java:219) ~[spring-beans-5.2.7.RELEASE.jar:5.2.7.RELEASE]
	at org.springframework.beans.factory.support.AbstractBeanFactory.doGetBean(AbstractBeanFactory.java:321) ~[spring-beans-5.2.7.RELEASE.jar:5.2.7.RELEASE]
	at org.springframework.beans.factory.support.AbstractBeanFactory.getBean(AbstractBeanFactory.java:202) ~[spring-beans-5.2.7.RELEASE.jar:5.2.7.RELEASE]
	at org.springframework.beans.factory.config.DependencyDescriptor.resolveCandidate(DependencyDescriptor.java:276) ~[spring-beans-5.2.7.RELEASE.jar:5.2.7.RELEASE]
	at org.springframework.beans.factory.support.DefaultListableBeanFactory.doResolveDependency(DefaultListableBeanFactory.java:1304) ~[spring-beans-5.2.7.RELEASE.jar:5.2.7.RELEASE]
	at org.springframework.beans.factory.support.DefaultListableBeanFactory.resolveDependency(DefaultListableBeanFactory.java:1224) ~[spring-beans-5.2.7.RELEASE.jar:5.2.7.RELEASE]
	at org.springframework.beans.factory.support.ConstructorResolver.resolveAutowiredArgument(ConstructorResolver.java:884) ~[spring-beans-5.2.7.RELEASE.jar:5.2.7.RELEASE]
	at org.springframework.beans.factory.support.ConstructorResolver.createArgumentArray(ConstructorResolver.java:788) ~[spring-beans-5.2.7.RELEASE.jar:5.2.7.RELEASE]
	at org.springframework.beans.factory.support.ConstructorResolver.autowireConstructor(ConstructorResolver.java:227) ~[spring-beans-5.2.7.RELEASE.jar:5.2.7.RELEASE]
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.autowireConstructor(AbstractAutowireCapableBeanFactory.java:1358) ~[spring-beans-5.2.7.RELEASE.jar:5.2.7.RELEASE]
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.createBeanInstance(AbstractAutowireCapableBeanFactory.java:1204) ~[spring-beans-5.2.7.RELEASE.jar:5.2.7.RELEASE]
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.doCreateBean(AbstractAutowireCapableBeanFactory.java:557) ~[spring-beans-5.2.7.RELEASE.jar:5.2.7.RELEASE]
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.createBean(AbstractAutowireCapableBeanFactory.java:517) ~[spring-beans-5.2.7.RELEASE.jar:5.2.7.RELEASE]
	at org.springframework.beans.factory.support.AbstractBeanFactory.lambda$doGetBean$0(AbstractBeanFactory.java:323) ~[spring-beans-5.2.7.RELEASE.jar:5.2.7.RELEASE]
	at org.springframework.beans.factory.support.DefaultSingletonBeanRegistry.getSingleton(DefaultSingletonBeanRegistry.java:226) ~[spring-beans-5.2.7.RELEASE.jar:5.2.7.RELEASE]
	at org.springframework.beans.factory.support.AbstractBeanFactory.doGetBean(AbstractBeanFactory.java:321) ~[spring-beans-5.2.7.RELEASE.jar:5.2.7.RELEASE]
	at org.springframework.beans.factory.support.AbstractBeanFactory.getBean(AbstractBeanFactory.java:202) ~[spring-beans-5.2.7.RELEASE.jar:5.2.7.RELEASE]
	at org.springframework.beans.factory.config.DependencyDescriptor.resolveCandidate(DependencyDescriptor.java:276) ~[spring-beans-5.2.7.RELEASE.jar:5.2.7.RELEASE]
	at org.springframework.beans.factory.support.DefaultListableBeanFactory.doResolveDependency(DefaultListableBeanFactory.java:1304) ~[spring-beans-5.2.7.RELEASE.jar:5.2.7.RELEASE]
	at org.springframework.beans.factory.support.DefaultListableBeanFactory.resolveDependency(DefaultListableBeanFactory.java:1224) ~[spring-beans-5.2.7.RELEASE.jar:5.2.7.RELEASE]
	at org.springframework.beans.factory.support.ConstructorResolver.resolveAutowiredArgument(ConstructorResolver.java:884) ~[spring-beans-5.2.7.RELEASE.jar:5.2.7.RELEASE]
	at org.springframework.beans.factory.support.ConstructorResolver.createArgumentArray(ConstructorResolver.java:788) ~[spring-beans-5.2.7.RELEASE.jar:5.2.7.RELEASE]
	at org.springframework.beans.factory.support.ConstructorResolver.autowireConstructor(ConstructorResolver.java:227) ~[spring-beans-5.2.7.RELEASE.jar:5.2.7.RELEASE]
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.autowireConstructor(AbstractAutowireCapableBeanFactory.java:1358) ~[spring-beans-5.2.7.RELEASE.jar:5.2.7.RELEASE]
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.createBeanInstance(AbstractAutowireCapableBeanFactory.java:1204) ~[spring-beans-5.2.7.RELEASE.jar:5.2.7.RELEASE]
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.doCreateBean(AbstractAutowireCapableBeanFactory.java:557) ~[spring-beans-5.2.7.RELEASE.jar:5.2.7.RELEASE]
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.createBean(AbstractAutowireCapableBeanFactory.java:517) ~[spring-beans-5.2.7.RELEASE.jar:5.2.7.RELEASE]
	at org.springframework.beans.factory.support.AbstractBeanFactory.lambda$doGetBean$0(AbstractBeanFactory.java:323) ~[spring-beans-5.2.7.RELEASE.jar:5.2.7.RELEASE]
	at org.springframework.beans.factory.support.DefaultSingletonBeanRegistry.getSingleton(DefaultSingletonBeanRegistry.java:226) ~[spring-beans-5.2.7.RELEASE.jar:5.2.7.RELEASE]
	at org.springframework.beans.factory.support.AbstractBeanFactory.doGetBean(AbstractBeanFactory.java:321) ~[spring-beans-5.2.7.RELEASE.jar:5.2.7.RELEASE]
	at org.springframework.beans.factory.support.AbstractBeanFactory.getBean(AbstractBeanFactory.java:202) ~[spring-beans-5.2.7.RELEASE.jar:5.2.7.RELEASE]
	at org.springframework.beans.factory.support.DefaultListableBeanFactory.preInstantiateSingletons(DefaultListableBeanFactory.java:893) ~[spring-beans-5.2.7.RELEASE.jar:5.2.7.RELEASE]
	at org.springframework.context.support.AbstractApplicationContext.finishBeanFactoryInitialization(AbstractApplicationContext.java:879) ~[spring-context-5.2.7.RELEASE.jar:5.2.7.RELEASE]
	at org.springframework.context.support.AbstractApplicationContext.refresh(AbstractApplicationContext.java:551) ~[spring-context-5.2.7.RELEASE.jar:5.2.7.RELEASE]
	at org.springframework.boot.SpringApplication.refresh(SpringApplication.java:758) ~[spring-boot-2.3.1.RELEASE.jar:2.3.1.RELEASE]
	at org.springframework.boot.SpringApplication.refresh(SpringApplication.java:750) ~[spring-boot-2.3.1.RELEASE.jar:2.3.1.RELEASE]
	at org.springframework.boot.SpringApplication.refreshContext(SpringApplication.java:397) ~[spring-boot-2.3.1.RELEASE.jar:2.3.1.RELEASE]
	at org.springframework.boot.SpringApplication.run(SpringApplication.java:315) ~[spring-boot-2.3.1.RELEASE.jar:2.3.1.RELEASE]
	at org.springframework.boot.SpringApplication.run(SpringApplication.java:1237) ~[spring-boot-2.3.1.RELEASE.jar:2.3.1.RELEASE]
	at org.springframework.boot.SpringApplication.run(SpringApplication.java:1226) ~[spring-boot-2.3.1.RELEASE.jar:2.3.1.RELEASE]
	at com.example.webdemo.WebDemoApplication.main(WebDemoApplication.java:15) ~[classes/:na]

2020-08-14 11:16:28.092 ERROR 34882 --- [           main] o.s.b.d.LoggingFailureAnalysisReporter   : 

***************************
APPLICATION FAILED TO START
***************************

Description:

The dependencies of some of the beans in the application context form a cycle:

┌─────┐
|  a defined in file [/Users/zhangyi/workspace/Study/web-demo/target/classes/com/example/webdemo/A.class]
↑     ↓
|  b defined in file [/Users/zhangyi/workspace/Study/web-demo/target/classes/com/example/webdemo/B.class]
└─────┘



## 核心方法分析
```java
	public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			// Prepare this context for refreshing.
			prepareRefresh();

			// Tell the subclass to refresh the internal bean factory.
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			// Prepare the bean factory for use in this context.
			prepareBeanFactory(beanFactory);

			try {
				// Allows post-processing of the bean factory in context subclasses.
				postProcessBeanFactory(beanFactory);

				// Invoke factory processors registered as beans in the context.
				invokeBeanFactoryPostProcessors(beanFactory);

				// Register bean processors that intercept bean creation.
				registerBeanPostProcessors(beanFactory);

				// Initialize message source for this context.
				initMessageSource();

				// Initialize event multicaster for this context.
				initApplicationEventMulticaster();

				// Initialize other special beans in specific context subclasses.
				onRefresh();

				// Check for listener beans and register them.
				registerListeners();

				// Instantiate all remaining (non-lazy-init) singletons.
				finishBeanFactoryInitialization(beanFactory);

				// Last step: publish corresponding event.
				finishRefresh();
			}

			catch (BeansException ex) {
				if (logger.isWarnEnabled()) {
					logger.warn("Exception encountered during context initialization - " +
							"cancelling refresh attempt: " + ex);
				}

				// Destroy already created singletons to avoid dangling resources.
				destroyBeans();

				// Reset 'active' flag.
				cancelRefresh(ex);

				// Propagate exception to caller.
				throw ex;
			}

			finally {
				// Reset common introspection caches in Spring's core, since we
				// might not ever need metadata for singleton beans anymore...
				resetCommonCaches();
			}
		}
	}
```



# Customizing the nature of a bean 自定义bean的特性
##  Lifecycle callbacks 生命周期回调
- Initialzation callbacks: 
	@PostConstruct, @Predestroy, InitializatingBean.afterPropertiesSet, DisposableBean.destroy

- LifeCycle
  (Note1: does not imply auto-startup at context refresh time)
  (Note: stop notifications are not guaranteed to come before destruction)

- SmartLifecycle: auto-startup, auto-close along with spring application context

applicationContext.registerShutdownHook: for non-web applications
## xxAware相关回调
- ApplicationContextAware, BeanNameAware
- BeanFactoryAware
- LoadTimeWeaverAware

## Bean definition inheritance
bean definition可以包含很多配置这个bean的信息，包括构造方法参数、属性值、容器特有信息（比如初始化方法，静态工厂方法名等）。
bean definition通过父子继承关系可以少敲很多键盘，等同于模板化。

## Container extension points 容器级别的扩展点

### BeanPostProcessor
有2个回调，一个在init前，一个在init后。
可以用来实现自己的初始化bean的逻辑（覆盖容器默认行为），依赖分析逻辑等等。
你可以植入一个或者多个BeanPostProcessor，以便在spring容器对bean完成实例化(instantiating)、配置(configuring)、初始化(initializing)后实现一些自定义的逻辑。
当有多个BeanPostProcessor时，你可以通过order设置处理顺序。

通常BeanPostProcessor可以用来做一些检查，还有一些AOP的类被实现为BeanPostProcessor以便封装代理逻辑。
典型用法：org.springframework.aop.framework.autoproxy.AbstractAutoProxyCreator
这个类根据实例对象创建proxy

* 注意
在用@Bean+Configuration class创建BeanPostProcessor的时候，返回类型必须是实现类，或者BeanPostProcessor类型，这样spring才好对这种bean特殊处理。
spring需要对BeanPostProcessor特殊处理：必须早点实例化他们，以便在其他bean初始化的时候调动他们。

* tips
虽然最推荐的用法是通过ApplicationContext的auto-detection自动检测来注册BeanPostProcessor，
你也可以用ConfigurableBeanFactory.addBeanPostProcessor手工注册BeanPostProcessor，但这样注册就不会遵循order顺序（总是在最前面调用）。


* auto-proxying
BeanPostProcessor的实现类会被spring容器特殊处理：
1、对应的实例、以及引用的普通实例  在容器启动的时候作为特殊的启动phase被初始化。
2、BeanPostProcessor实例被注册到容器中，以便后续作用在其他普通bean上。
3、由于自动代理auto-proxying也是在BeanPostProcessor中实现的，所以BeanPostProcessor本身不能被auto-proxying，不能植入切面，
如有违反，你会看到这样的日志：
Bean someBean is not eligible for getting processed by all BeanPostProcessor interfaces (for example: not eligible for auto-proxying).
 
如果你的bean通过@Resource自动注入到BeanPostProcessor，但是又通过name匹配失败了，spring会fallback走type按类型匹配
因为类型匹配依赖 访问非预期的beans，因此会让这些bean无法被auto-proxying或者被其他BeanPostProcessor处理。

经验：尽量避免BeanPostProcessor使用自动注入。

### BeanFactoryPostProcessor 自定义配置元数据
BeanfactoryPostProcessor 的语义和 BeanPostProcessor类似，最大的不同在于：
BeanfactoryPostProcessor 操作的是 bean configuration metadata 元数据
他可以读取、修改元数据，使得他能在 **实例化** 之前修改bean元数据。

如果你想修改bean实例（通过元数据配置的），那么你需要转而使用BeanPostProcessor。
虽然从技术上来讲你可以用BeanFactoryPostProcessor与bean实例一起工作（比如BeanFactory.getBean())，
这会bean的导致提前初始化，违反了容器的标准生命周期，从而导致负作用，比如：穿透bean post processing。

另外，BeanFactoryPostProcessor实例是容器范围的（如果你没用多个容器）。

为了对bean元数据进行修改，spring会自动运行 BeanFactoryPostProcessor
spring包含了一系列预定义的 BeanFactoryPostProcessor:
- PropertyOverrideConfigurer
- PropertySourcesPlaceholderConfigurer

与BeanPostProcessors一样，BeanFactoryPostProcessor设置lazy init没意义。

### FactoryBean 自定义实例化逻辑
FactoryBean是spring容器实例化bean的一个扩展点。
如果你有复杂的实例化逻辑，你可以用他来实现，spring用到的地方非常多，比如：
- MBeanProxyFactoryBean
- JpaRepositoryFactoryBean
- MethodInvokingFactoryBean
- TaskExecutorFactoryBean
- ConversionServiceFactoryBean
- TransactionProxyFactoryBean




# Spring启动过程

component scan package

register bean definitions

create bean object

bean name aware

bean post processor: init/afterPropertiesSet

inject beans




# Bean生命周期

# Spring启停回调


# Spring ApplicationContext容器生命周期
doClose 


# Spring 如何实现依赖bean加载？如何实现依赖bean的释放？
