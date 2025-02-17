---
layout: post
title: "Spring Boot 中通过 FactoryBean 进行 Bean 的实例化"
date: 2024-03-03 10:00:00 +0800
tags: [Jekyll, Markdown, 博客]
author: "孙珂"
---


# Spring Boot 中通过 FactoryBean 进行 Bean 的实例化

## Bean的实例化

Spring 中实例化 Bean 的方法有很多，如通过 @Component 注解、@Bean 注解、FactoryBean 接口等。

通过 FactoryBean 接口实例化 Bean 的好处是可以将复杂的实例化逻辑封装在相应的实现类中，使得配置更加简洁。

FactoryBean 的抽象类 AbstractFactoryBean 对 FactoryBean 进一步进行了封装，使其具备完整的 Bean 生命周期管理能力。


## 使用 FactoryBean 和 AbstractFactoryBean 分别进行 Bean 的实例化

首先定义一个类

```java

@Data
public class AlbumService {
    public AlbumService() {
        System.out.println("AlbumService init");
    }

    public String name = "AlbumService";

}


```

### 使用 FactoryBean 进行 Bean 的实例化

FactoryBean 接口比较简单，只有 getObject()，getObjectType() 和 isSingleton() 三个方法，其中 isSingleton() 为 default 实现。

创建一个实现 FactoryBean 接口的类 MyAlbumFactoryBean，实现 getObject 和 getObjectType 方法。

getObject 返回期望的 AlbumService 对象。


```java
public class MyAlbumFactoryBean implements FactoryBean<AlbumService> {
    @Override
    public AlbumService getObject() throws Exception {
        AlbumService myAlbumService = new AlbumService();
        myAlbumService.setName("AlbumServiceV1");
        return myAlbumService;
    }

    @Override
    public Class<?> getObjectType() {
        return AlbumService.class;
    }
}
```

接着使用 @Bean 注解将 MyAlbumFactoryBean 注册到 Spring 容器中。

```java
@Configuration
public class AppConfig {

    @Bean
    public MyAlbumFactoryBean myAlbumFactoryBean() {
        return new MyAlbumFactoryBean();
    }
}

```

再通过 BeanFactory 来获取相应的 AlbumService，getBean("myAlbumFactoryBean") 拿到 MyAlbumFactoryBean 的 getObject 方法返回的 AlbumService 对象。

```java

@Autowired
private ApplicationContext applicationContext;

public void getBean() {
    BeanFactory beanFactory = applicationContext;
    AlbumService albumService = (AlbumService) applicationContext.getBean("myAlbumFactoryBean");
}
```


### 使用 AbstractFactoryBean 进行 Bean 的实例化

AbstractFactoryBean 对 FactoryBean 进行了封装，
并且实现了 FactoryBean<T>, BeanClassLoaderAware, BeanFactoryAware, InitializingBean, DisposableBean这些接口，
也就是这些接口，使其具备完整的 Bean 生命周期管理能力。

AbstractFactoryBean 通过 createInstance 封装了 FactoryBean 的 getObject 方法。

同过 @Override afterPropertiesSet 方法，对 Bean 的生命周期进行管理。（上面几个接口的实现都做了类似的事情，如在创建阶段，销毁阶段进行管理）


```java

public class MyAlbumAbstractFactoryBean extends AbstractFactoryBean<AlbumService> {


    public String toCheckName;

    @Override
    public void afterPropertiesSet() throws Exception {
        super.afterPropertiesSet();
        System.out.println("afterPropertiesSet doSomething");
        if (toCheckName == null) {
            System.out.println("toCheckName is null");
        } else {
            System.out.println("toCheckName is not null" + toCheckName);
        }
    }

    @NotNull
    @Override
    protected AlbumService createInstance() throws Exception {
        System.out.println("createInstance" + this.getBeanFactory());
        AlbumService myAlbumService = new AlbumService();
        myAlbumService.setName("MyAlbumAbstractFactoryBean");
        return myAlbumService;
    }

    @Override
    public Class<?> getObjectType() {
        return MyAlbumComponent.class;
    }

    public void setToCheckName(String toCheckName) {
        this.toCheckName = toCheckName;
    }
}


```


通过 @Bean 注解将 MyAlbumAbstractFactoryBean 注册到 Spring 容器中。


```java

@Configuration
public class AppConfig {


    @Bean
    public MyAlbumAbstractFactoryBean myAlbumAbstractFactoryBean() {
        MyAlbumAbstractFactoryBean myAlbumAbstractFactoryBean = new MyAlbumAbstractFactoryBean();
        myAlbumAbstractFactoryBean.setSingleton(true);
        myAlbumAbstractFactoryBean.setToCheckName("name checked");
        return myAlbumAbstractFactoryBean;
    }

}


```

和上面的 FactoryBean 一样，通过 BeanFactory 来获取相应的 AlbumService，getBean("myAlbumFactoryBean") 拿到 MyAlbumAbstractFactoryBean 返回的 AlbumService 对象。


```java


        @Autowired
        private ApplicationContext applicationContext;

        public void getBean() {

            AlbumService myAbstractFactoryBeanAlbumService = (AlbumService) applicationContext.getBean("myAlbumAbstractFactoryBean");
            System.out.println("myAbstractFactoryBeanAlbum " + myAbstractFactoryBeanAlbumService);
        }

```

## 灵活实现

可以结合 FactoryBean , BeanClassLoaderAware, BeanFactoryAware, InitializingBean, DisposableBean这些接口，
根据需要进行组合，来获取 IoC 容器管理 Bean 的生命周期的能力。

```java
public class MyAlbumFactoryBean implements FactoryBean<AlbumService>, InitializingBean {
    @Override
    public AlbumService getObject() throws Exception {
        AlbumService myAlbumService = new AlbumService();
        myAlbumService.setName("AlbumServiceV1");
        return myAlbumService;
    }

    @Override
    public Class<?> getObjectType() {
        return AlbumService.class;
    }

    @Override
    public void afterPropertiesSet() throws Exception {
        System.out.println("MyAlbumFactoryBean#afterPropertiesSet");
    }
}
```

根据上面的初始化管理，可以测试一下常见的几种初始化方法的执行顺序，依次是@PostConstruct，afterPropertiesSet，@Bean(initMethod = "xxx")

```java
public class MyAccountService implements InitializingBean {
    @PostConstruct
    public void init() {
        System.out.println("MyAccountService PostConstruct first init");
    }

    @Override
    public void afterPropertiesSet() throws Exception {
        System.out.println("MyAccountService afterPropertiesSet second init");
    }

    public void initV1(){
        System.out.println("MyAccountService initMethod third init");
    }
}


@Bean(initMethod = "initV1")
public MyAccountService myAccountService() {
    return new MyAccountService();
}

```


## 结论

通过 FactoryBean 可以对复杂的创建逻辑从@Bean中提取出来，进行封装。

AbstractFactoryBean 在 FactoryBean 的基础上，进一步扩展，使其具备完整的 Bean 生命周期管理能力。