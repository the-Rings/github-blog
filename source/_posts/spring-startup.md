---
title: SpringBoot启动分析
date: 2021-11-05 18:43:05
categories:
- Spring
---

SpringBoot的启动过程就是SpringApplication实例化并运行run方法的过程，最终返回ConfigurableApplicationContext实例。 ConfigurableApplicationContext继承了ApplicationContext。

1. 获取监听器，启动事件监听
2. 获取命令行参数，准备项目环境
3. 打印Banner
4. 创建ApplicationContext，对于Web应用返回AnnotationConfigServletWebServerApplicationContext
5. prepareContext
6. refreshContext
7. afterRefresh, 三个方法完整的建立Context
8. 向监听器发出执行结束的通知，返回ConfigurableApplicationContext

附上run方法源码:
```java
/**
 * Run the Spring application, creating and refreshing a new
 * {@link ApplicationContext}.
 * @param args the application arguments (usually passed from a Java main method)
 * @return a running {@link ApplicationContext}
 */
public ConfigurableApplicationContext run(String... args) {
	long startTime = System.nanoTime();
	DefaultBootstrapContext bootstrapContext = createBootstrapContext();
	ConfigurableApplicationContext context = null;
	configureHeadlessProperty();
	SpringApplicationRunListeners listeners = getRunListeners(args);
	listeners.starting(bootstrapContext, this.mainApplicationClass);                                                 // [1]
	try {
		ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
		ConfigurableEnvironment environment = prepareEnvironment(listeners, bootstrapContext, applicationArguments); // [2]
		configureIgnoreBeanInfo(environment);
		Banner printedBanner = printBanner(environment);                                                             // [3]
		context = createApplicationContext();                                                                        // [4]
		context.setApplicationStartup(this.applicationStartup);
		prepareContext(bootstrapContext, context, environment, listeners, applicationArguments, printedBanner);      // [5]
		refreshContext(context);                                                                                     // [6]
		afterRefresh(context, applicationArguments);                                                                 // [7]
		Duration timeTakenToStartup = Duration.ofNanos(System.nanoTime() - startTime);
		if (this.logStartupInfo) {
			new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), timeTakenToStartup);
		}
		listeners.started(context, timeTakenToStartup);                                                              // [8]
		callRunners(context, applicationArguments);
	}
	catch (Throwable ex) {
		handleRunFailure(context, ex, listeners);
		throw new IllegalStateException(ex);
	}
	try {
		Duration timeTakenToReady = Duration.ofNanos(System.nanoTime() - startTime);
		listeners.ready(context, timeTakenToReady);
	}
	catch (Throwable ex) {
		handleRunFailure(context, ex, null);
		throw new IllegalStateException(ex);
	}
	return context;
}
```


