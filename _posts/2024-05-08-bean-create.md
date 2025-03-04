---
layout: post
title: "Spring Boot 通过 BeanPostProcessor 来控制 Bean 的创建"
date: 2024-05-08 10:00:00 +0800
tags: [ Jekyll, Markdown, 博客 ]
author: "孙珂"
---

#  Spring Boot 通过 BeanPostProcessor 来控制 Bean 的创建

## createBean 的执行流程

createBean 是 Spring Boot 创建 Bean 的核心方法。

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

hasInstantiationAwareBeanPostProcessors() 就是判断是否有相应的 实现

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

BeanPostProcessor(InstantiationAwareBeanPostProcessor) 的 postProcessBeforeInstantiation，postProcessAfterInstantiation 是控制 Bean 的实例化前后的动作。