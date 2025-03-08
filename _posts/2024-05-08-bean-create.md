---
layout: post
title: "Spring Boot 通过 BeanPostProcessor 来控制 Bean 的创建"
date: 2024-05-08 10:00:00 +0800
tags: [ Jekyll, Markdown, 博客 ]
author: "孙珂"
---

#  Spring Boot 中 Bean 的创建和销毁

## createBean 的执行流程

createBean 是 Spring Boot 创建 Bean 的核心方法。

## BeanPostProcessor

自定义 Bean，实现了 InitializingBean，DisposableBean 接口

```java
@Service
public class MyHomeService implements InitializingBean, DisposableBean {
    @PostConstruct
    public void postConstruct() {
        System.out.println("MyHomeService PostConstruct first init 2");
    }

    @Override
    public void afterPropertiesSet() throws Exception {
        System.out.println("MyHomeService afterPropertiesSet second init 3");
    }

    public void initFunction() {
        System.out.println("MyHomeService initMethod third init");
    }

    /**
     * ide find Usages
     */
    @PreDestroy
    public void preDestroy() {
        System.out.println("MyHomeService DisposableBean first destroy 4");
    }

    @Override
    public void destroy() throws Exception {
        System.out.println("MyHomeService PreDestroy#destroy second destroy 5");

    }

    public void destroyFunction() {
        System.out.println("MyHomeService PreDestroy third destroy");
    }
}

```
自定义 MyHomeServiceBeanPostProcessor 实现了 BeanPostProcessor 接口

注意 需要和 MyHomeService 分开定义
```java
@Component
public class MyHomeServiceBeanPostProcessor implements BeanPostProcessor {
    @Override
    public  Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        if(ObjectUtils.nullSafeEquals(beanName,"myHomeService")){
            System.out.println("MyHomeServiceHandler postProcessBeforeInitialization 1");
        }
        return bean;
    }


    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        if(ObjectUtils.nullSafeEquals(beanName,"myHomeService")){
            System.out.println("MyHomeServiceHandler postProcessAfterInitialization 4");
        }
        return bean;
    }

}


```

DestructionAwareBeanPostProcessor 接口，处理 DestructionAware 

```java
@Component
public class MyHomeServiceDestructionAwarePostProcessor implements DestructionAwareBeanPostProcessor {


    @Override
    public void postProcessBeforeDestruction(Object bean, String beanName) throws BeansException {
        if(ObjectUtils.nullSafeEquals(beanName,"myHomeService")){
            System.out.println("MyHomeService DestructionAwareBeanPostProcessor 1");
        }
    }
}
```
通过日志，可以看到初始化方法和销毁方法的调用顺序
```
MyHomeServiceHandler postProcessBeforeInitialization 1
MyHomeService PostConstruct first init 2
MyHomeService afterPropertiesSet second init 3
MyHomeServiceHandler postProcessAfterInitialization 4

MyHomeService DestructionAwareBeanPostProcessor 1
MyHomeService PreDestroy destroy 2
MyHomeService DisposableBean#destroy destroy 3
```

## InstantiationAwareBeanPostProcessor

InstantiationAwareBeanPostProcessor 是 BeanPostProcessor 的子接口。

其中通过对是否有 InstantiationAwareBeanPostProcessor 相关实现的判断，来控制 Bean 的创建。

如果存在 InstantiationAwareBeanPostProcessor 就通过 InstantiationAwareBeanPostProcessor 来处理，并返回结果 postProcessBeforeInstantiation() 这个方法实现的自定义结果

如果不存在相应的 InstantiationAwareBeanPostProcessor 就执行 doCreateBean()，使用 Spring 默认的方式创建 Bean 。


```java
	/**
     * org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#createBean
	 * Central method of this class: creates a bean instance,
	 * populates the bean instance, applies post-processors, etc.
	 */
	@Override
	protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
			throws BeanCreationException {

		try {
			// Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
			Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
			if (bean != null) {
				return bean;
			}
		}
		catch (Throwable ex) {
		}

		try {
			Object beanInstance = doCreateBean(beanName, mbdToUse, args);
			if (logger.isTraceEnabled()) {
				logger.trace("Finished creating instance of bean '" + beanName + "'");
			}
			return beanInstance;
		}
        catch (Throwable ex) {
            ...
        }
        ...
	}
```

hasInstantiationAwareBeanPostProcessors() 就是判断是否有相应的实现

```java
	/**
     * org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#resolveBeforeInstantiation
     *
	 * Apply before-instantiation post-processors, resolving whether there is a
	 * before-instantiation shortcut for the specified bean.
	 */
	@Nullable
	protected Object resolveBeforeInstantiation(String beanName, RootBeanDefinition mbd) {
		Object bean = null;
		if (!Boolean.FALSE.equals(mbd.beforeInstantiationResolved)) {
			// Make sure bean class is actually resolved at this point.
			if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
				Class<?> targetType = determineTargetType(beanName, mbd);
				if (targetType != null) {
					bean = applyBeanPostProcessorsBeforeInstantiation(targetType, beanName);
					if (bean != null) {
						bean = applyBeanPostProcessorsAfterInitialization(bean, beanName);
					}
				}
			}
			mbd.beforeInstantiationResolved = (bean != null);
		}
		return bean;
	}


```

## 自定义 InstantiationAwareBeanPostProcessor 来控制 Bean 的创建

postProcessBeforeInstantiation 可以在 Bean 实例化之前进行拦截，返回代理对象。存在远程RPC调用的场景可以这样返回。

postProcessAfterInstantiation 可以在 Bean 实例化之后进行拦截，返回 false ，spring boot 不进行属性的赋值，可以自己组装属性。返回 true 就是原有逻辑。

```java
@Service
public class MyAddressInstantiationAwareBeanPostProcessor implements InstantiationAwareBeanPostProcessor {


    /**
     * 远程RPC调用 可以用这个方法进行拦截 返回代理对象
     */
    @Override
    public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException {
        if (ObjectUtils.nullSafeEquals("myAddressHandler", beanName) && MyAddressHandler.class.equals(beanClass)) {
            System.out.println("MyBeanDefinitionService  myAddressHandlerBefore bean is created");
            MyAddressHandler myAddressHandler = new MyAddressHandler();
            return myAddressHandler;
        }
        return null;
    }

    /**
     * Bean 实例化后操作 
     * 返回false 不进行属性的赋值 
     * 判断Bean应不应该赋值 可以自己组装 类似拦截机制 Bean的实例化完成了 属性的赋值没有完成
     * true 原有逻辑
     * false 跳过属性植入
     */
    @Override
    public boolean postProcessAfterInstantiation(Object bean, String beanName) throws BeansException {
        if (ObjectUtils.nullSafeEquals("myAddressHandler", beanName) && MyAddressHandler.class.equals(bean.getClass())) {
            System.out.println("MyBeanDefinitionService postProcessAfterInstantiation bean is created");
            MyAddressHandler myAddressHandler = (MyAddressHandler) bean;
            myAddressHandler.setAddress("new address");
            return false;
        }
        return true;
    }
}

```

## 总结

InitializingBean, DisposableBean 接口是控制 Bean 的初始化和销毁的。

实现了 BeanPostProcessor 那么所有 Bean 的创建都会回调对应的方法，需要判断是否是相应的 Bean 再进行处理。

InstantiationAwareBeanPostProcessor 的 postProcessBeforeInstantiation，postProcessAfterInstantiation 是控制 Bean 的实例化( instantiate )前后的动作。

如果被 InstantiationAwareBeanPostProcessor 接管了，@PostConstruct, InitializingBean, @PreDestroy, DisposableBean 这些注解就不会生效。(因为没有被 Spring-boot 容器管理)