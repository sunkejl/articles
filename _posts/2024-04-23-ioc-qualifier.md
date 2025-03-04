---
layout: post
title: "对 Spring Boot 依赖注入进行自定义分组"
date: 2024-04-23 10:00:00 +0800
tags: [ Jekyll, Markdown, 博客 ]
author: "孙珂"
---

# 对 Spring Boot 依赖注入进行自定义分组

## 通过 @Qualifier 注解可以指定 Bean 名称或 Id 进行注入
在依赖注入的时候，通过 @Qualifier 注解可以指定 Bean 名称或 Id 进行注入，

这样存在多个 Type 类型相同的 Bean 的时候，可以进行区分。

```java

@Autowired
@Qualifier("myAlbumService")//指定 Bean 名称或 Id
private AlbumService albumService;
```


## 通过 @Qualifier 注解可以进行对 Bean 的分组

@Qualifier 还可以进行对 Bean 的分组，默认会对添加了 @Qualifier 注解的 Bean 进行分组。


因为 @Qualifier 有 通过自定义注解 @MyFavorite 实现对注解 @Qualifier 的扩展

```java

@Target({ElementType.FIELD, ElementType.METHOD, ElementType.PARAMETER, ElementType.TYPE, ElementType.ANNOTATION_TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
@Qualifier
public @interface MyFavorite {
}
```

@MyFavorite 实现了对 @Qualifier 的继承，从而达到了 Bean 分组的目的

```java

public interface Coffee {
    String getName();
}


@Service
@MyFavorite
public class Americano implements Coffee {
    @Override
    public String getName() {
        return "Americano";
    }
}

@Service
@MyFavorite
public class Cappuccino implements Coffee {
    @Override
    public String getName() {
        return "Cappuccino";
    }
}


@Service
public class Espresso implements Coffee{
    @Override
    public String getName() {
        return "Espresso";
    }
}


@Service
public class Latte implements Coffee {
    @Override
    public String getName() {
        return "Latte";
    }
}


```


这样在依赖注入 (Dependency Injection) 的时候，就可以根据分组进行注入

通过字段进行注入

```java
    @Autowired
    private List<Coffee> coffeeList;

    @Autowired
    @MyFavorite
    private List<Coffee> myFavoriteCoffeeList;
```



通过构造器进行注入

```java

@Service
@Getter
public class MyFavoriteCoffeeService {
    private List<Coffee> coffeeList;

    public MyFavoriteCoffeeService(@MyFavorite List<Coffee> coffeeList) {
        this.coffeeList = coffeeList;
    }
}


```

## 结论

@Qualifier 注解不仅可以指定 Bean 名称或 Id 进行注入，还可以进行对 Bean 的分组，通过自定义的扩展，使分组更加可读。

## 相关注入Tips

1. 通过 @Autowire 字段注入虽然便利，但有外部依赖，不那么纯粹。

2. 通过构造器注入，具有完整的初始化状态，低依赖，保证强制注入，避免状态被修改，不过参数过多就是坏味道(bad code smell)。

3. @Autowire Spring 启动时就会实例化它。

4. ObjectProvider 懒加载，Spring 会把它注册为 Bean，但不会立即创建它，等到使用的时候才进行实例化。(org.springframework.beans.factory.support.DefaultListableBeanFactory#resolveDependency)

5. 相关配置，多用枚举，提高代码的可读性。
```java
   @Bean("configUserBean")
   @Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)
   ```

