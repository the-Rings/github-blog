---
title: Spring Aware
date: 2023-02-04 12:55:47
tags: Aware
categories: 
- Spring
---

Aware接口的作用
Spring IOC容器在创建Bean对象的过程中，如果还需要其他对象，此时可以将对象实现Aware接口，来满足需要。
比如，我要通过Foo的对象来获得ApplicationContext，举例如下（XML配置省略）：
```Java
class Foo implements ApplicationContextAware {
	private ApplicationContext applicationContext;
	private String name;
	@Override
	public void setApplicationContext(ApplicationContext applicationContext) {
		this.applicationContext = applicationContext;
	}

	public ApplicationContext getApplicationContext() {
		return this.applicationContext;
	}
}

public class MyAwareTest {
	public static void public static void main(String[] args) {
		ApplicationContext applicationContext = new ClassPathXmlApplication("YOUR_XML_PATH");
		Foo fooBean = applicationContext.getBean(Foo.class);
		System.out.println(bean.getApplicationContext());
	}
}
```
`org.springframework.context.ApplicationContextAware`继承`org.springframework.beans.factory.Aware`接口，它是一个顶级Aware接口，作用仅仅是Marker（标记），各种Aware功能的接口都要继承。
在实现Aware时，需要借助AOP，于是就有了`ApplicationContextAwareProcessor`，可以看到它实现了`BeanPostProcessor`，`BeanPostProcessor`是在Bean对象实例化之后进行Bean的Processor（增强）。
```Java
class ApplicationContextAwareProcessor implements BeanPostProcessor {

	private final ConfigurableApplicationContext applicationContext;
    // code ...

	@Override
	@Nullable
	public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
		// code ...

		invokeAwareInterfaces(bean);
		return bean;
	}

	private void invokeAwareInterfaces(Object bean) {
		if (bean instanceof Aware) {
			if (bean instanceof EnvironmentAware environmentAware) {
				environmentAware.setEnvironment(this.applicationContext.getEnvironment());
			}
			if (bean instanceof EmbeddedValueResolverAware embeddedValueResolverAware) {
				embeddedValueResolverAware.setEmbeddedValueResolver(this.embeddedValueResolver);
			}
			if (bean instanceof ResourceLoaderAware resourceLoaderAware) {
				resourceLoaderAware.setResourceLoader(this.applicationContext);
			}
			if (bean instanceof ApplicationEventPublisherAware applicationEventPublisherAware) {
				applicationEventPublisherAware.setApplicationEventPublisher(this.applicationContext);
			}
			if (bean instanceof MessageSourceAware messageSourceAware) {
				messageSourceAware.setMessageSource(this.applicationContext);
			}
			if (bean instanceof ApplicationStartupAware applicationStartupAware) {
				applicationStartupAware.setApplicationStartup(this.applicationContext.getApplicationStartup());
			}
			if (bean instanceof ApplicationContextAware applicationContextAware) {
				applicationContextAware.setApplicationContext(this.applicationContext);
			}
		}
	}

}

```
