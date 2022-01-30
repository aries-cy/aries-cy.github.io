---
title: SpringBoot的启动流程
categories: [springboot]
comments: true
---

##### springboot的启动流程
1.入口，SpringBoot工程的启动类
```java
@SpringBootApplication
public class WebApplication {

    public static void main(String[] args) {
        SpringApplication.run(WebApplication.class, args);
    }

}
```
如上代码，在启动类上面加上`@SpringBootApplication`注解，在main()方法中调用`SpringApplication.run(WebApplication.class, args);`启动SpringBoot工程。

2.@SpringBootApplication注解

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = { @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
		@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {
    //...
}
```
可以看到`@SpringBootApplication`是一个组合注解，主要由`@SpringBootConfiguration`、`@EnableAutoConfiguration`和`@ComponentScan`这三个注解组合而成，即`@SpringBootApplication`主要的作用是作为一个配置类，能触发包扫描和自动装配，让与SpringBoot相关的bean被注入到spring容器中。

3.启动流程
点击进入SpringBoot启动类的main()方法，查看源代码：

```java
public static ConfigurableApplicationContext run(Class<?> primarySource, String... args) {
		return run(new Class<?>[] { primarySource }, args);
	}
```

上面的static run()方法又继续调用另一个静态run()方法：
```java
public static ConfigurableApplicationContext run(Class<?>[] primarySources, String[] args) {
		return new SpringApplication(primarySources).run(args);
	}
```
可以看到构建了一个`SpringApplication`对象，并调用其run()方法，来启动SpringBoot。
首先来看它的run()方法：
```java
// SpringApplication.java

public ConfigurableApplicationContext run(String... args) {
	// new 一个StopWatch用于统计run启动过程花了多少时间
	StopWatch stopWatch = new StopWatch();
	// 开始计时
	stopWatch.start();
	ConfigurableApplicationContext context = null;
	// exceptionReporters集合用来存储异常报告器，用来报告SpringBoot启动过程的异常
	Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
	// 配置headless属性，即“java.awt.headless”属性，默认为ture
	// 其实是想设置该应用程序,即使没有检测到显示器,也允许其启动.对于服务器来说,是不需要显示器的,所以要这样设置.
	configureHeadlessProperty();
	// 【1】从spring.factories配置文件中加载到EventPublishingRunListener对象并赋值给SpringApplicationRunListeners
	// EventPublishingRunListener对象主要用来发射SpringBoot启动过程中内置的一些生命周期事件，标志每个不同启动阶段
	SpringApplicationRunListeners listeners = getRunListeners(args);
	// 启动SpringApplicationRunListener的监听，表示SpringApplication开始启动。
	// 》》》》》发射【ApplicationStartingEvent】事件
	listeners.starting();
	try {
		// 创建ApplicationArguments对象，封装了args参数
		ApplicationArguments applicationArguments = new DefaultApplicationArguments(
				args);
		// 【2】准备环境变量，包括系统变量，环境变量，命令行参数，默认变量，servlet相关配置变量，随机值，
		// JNDI属性值，以及配置文件（比如application.properties）等，注意这些环境变量是有优先级的
		// 》》》》》发射【ApplicationEnvironmentPreparedEvent】事件
		ConfigurableEnvironment environment = prepareEnvironment(listeners,
				applicationArguments);
		// 配置spring.beaninfo.ignore属性，默认为true，即跳过搜索BeanInfo classes.
		configureIgnoreBeanInfo(environment);
		// 【3】控制台打印SpringBoot的bannner标志
		Banner printedBanner = printBanner(environment);
		// 【4】根据不同类型创建不同类型的spring applicationcontext容器
		// 因为这里是servlet环境，所以创建的是AnnotationConfigServletWebServerApplicationContext容器对象
		context = createApplicationContext();
		// 【5】从spring.factories配置文件中加载异常报告期实例，这里加载的是FailureAnalyzers
		// 注意FailureAnalyzers的构造器要传入ConfigurableApplicationContext，因为要从context中获取beanFactory和environment
		exceptionReporters = getSpringFactoriesInstances(
				SpringBootExceptionReporter.class,
				new Class[] { ConfigurableApplicationContext.class }, context); // ConfigurableApplicationContext是AnnotationConfigServletWebServerApplicationContext的父接口
		// 【6】为刚创建的AnnotationConfigServletWebServerApplicationContext容器对象做一些初始化工作，准备一些容器属性值等
		// 1）为AnnotationConfigServletWebServerApplicationContext的属性AnnotatedBeanDefinitionReader和ClassPathBeanDefinitionScanner设置environgment属性
		// 2）根据情况对ApplicationContext应用一些相关的后置处理，比如设置resourceLoader属性等
		// 3）在容器刷新前调用各个ApplicationContextInitializer的初始化方法，ApplicationContextInitializer是在构建SpringApplication对象时从spring.factories中加载的
		// 4）》》》》》发射【ApplicationContextInitializedEvent】事件，标志context容器被创建且已准备好
		// 5）从context容器中获取beanFactory，并向beanFactory中注册一些单例bean，比如applicationArguments，printedBanner
		// 6）TODO 加载bean到application context，注意这里只是加载了部分bean比如mainApplication这个bean，大部分bean应该是在AbstractApplicationContext.refresh方法中被加载？这里留个疑问先
		// 7）》》》》》发射【ApplicationPreparedEvent】事件，标志Context容器已经准备完成
		prepareContext(context, environment, listeners, applicationArguments,
				printedBanner);
		// 【7】刷新容器，这一步至关重要，以后会在分析Spring源码时详细分析，主要做了以下工作：
		// 1）在context刷新前做一些准备工作，比如初始化一些属性设置，属性合法性校验和保存容器中的一些早期事件等；
		// 2）让子类刷新其内部bean factory,注意SpringBoot和Spring启动的情况执行逻辑不一样
		// 3）对bean factory进行配置，比如配置bean factory的类加载器，后置处理器等
		// 4）完成bean factory的准备工作后，此时执行一些后置处理逻辑，子类通过重写这个方法来在BeanFactory创建并预准备完成以后做进一步的设置
		// 在这一步，所有的bean definitions将会被加载，但此时bean还不会被实例化
		// 5）执行BeanFactoryPostProcessor的方法即调用bean factory的后置处理器：
		// BeanDefinitionRegistryPostProcessor（触发时机：bean定义注册之前）和BeanFactoryPostProcessor（触发时机：bean定义注册之后bean实例化之前）
		// 6）注册bean的后置处理器BeanPostProcessor，注意不同接口类型的BeanPostProcessor；在Bean创建前后的执行时机是不一样的
		// 7）初始化国际化MessageSource相关的组件，比如消息绑定，消息解析等
		// 8）初始化事件广播器，如果bean factory没有包含事件广播器，那么new一个SimpleApplicationEventMulticaster广播器对象并注册到bean factory中
		// 9）AbstractApplicationContext定义了一个模板方法onRefresh，留给子类覆写，比如ServletWebServerApplicationContext覆写了该方法来创建内嵌的tomcat容器
		// 10）注册实现了ApplicationListener接口的监听器，之前已经有了事件广播器，此时就可以派发一些early application events
		// 11）完成容器bean factory的初始化，并初始化所有剩余的单例bean。这一步非常重要，一些bean postprocessor会在这里调用。
		// 12）完成容器的刷新工作，并且调用生命周期处理器的onRefresh()方法，并且发布ContextRefreshedEvent事件
		refreshContext(context);
		// 【8】执行刷新容器后的后置处理逻辑，注意这里为空方法
		afterRefresh(context, applicationArguments);
		// 停止stopWatch计时
		stopWatch.stop();
		// 打印日志
		if (this.logStartupInfo) {
			new StartupInfoLogger(this.mainApplicationClass)
					.logStarted(getApplicationLog(), stopWatch);
		}
		// 》》》》》发射【ApplicationStartedEvent】事件，标志spring容器已经刷新，此时所有的bean实例都已经加载完毕
		listeners.started(context);
		// 【9】调用ApplicationRunner和CommandLineRunner的run方法，实现spring容器启动后需要做的一些东西比如加载一些业务数据等
		callRunners(context, applicationArguments);
	}
	// 【10】若启动过程中抛出异常，此时用FailureAnalyzers来报告异常
	// 并》》》》》发射【ApplicationFailedEvent】事件，标志SpringBoot启动失败
	catch (Throwable ex) {
		handleRunFailure(context, ex, exceptionReporters, listeners);
		throw new IllegalStateException(ex);
	}

	try {
		// 》》》》》发射【ApplicationReadyEvent】事件，标志SpringApplication已经正在运行即已经成功启动，可以接收服务请求了。
		listeners.running(context);
	}
	// 若出现异常，此时仅仅报告异常，而不会发射任何事件
	catch (Throwable ex) {
		handleRunFailure(context, ex, exceptionReporters, null);
		throw new IllegalStateException(ex);
	}
	// 【11】最终返回容器
	return context;
}
```

以上代码就是SpringBoot的启动流程了，注释也比较详细，其中【x】记录了主要的步骤。
大概记录了SpringBoot的启动流程，并未深入的去深究细节，对SpringBoot的启动流程有了一个大概的整体意识。