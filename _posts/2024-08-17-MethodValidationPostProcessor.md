---
layout: post
title: "@Valid 和 @Validated 的区别"
date: 2024-08-12 10:00:00 +0800
tags: [ Jekyll, Markdown, 博客 ]
author: "孙珂"
---

# Spring 如何通过 @Validated 进行参数校验

## MethodValidationPostProcessor 和 MethodValidationInterceptor 的处理过程

Spring 中，和 validation 相关的类如下

org.springframework.validation.beanvalidation.MethodValidationPostProcessor 来实现参数校验的处理器。

org.springframework.validation.beanvalidation.MethodValidationInterceptor 用来处理校验方面的逻辑(AOP)。

通过错误日志，可以看到处理验证的过程。

```java
java.lang.NullPointerException: page is marked non-null but is null
at com.eknus.think.into.spring.controller.ValidController.getList(ValidController.java:31)
...
at org.springframework.validation.beanvalidation.MethodValidationInterceptor.invoke(MethodValidationInterceptor.java:123)
...
at org.springframework.aop.framework.CglibAopProxy$DynamicAdvisedInterceptor.intercept(CglibAopProxy.java:707)
at com.eknus.think.into.spring.controller.ValidController$$EnhancerBySpringCGLIB$$198b7ff3.getList(<generated>)
...
at java.lang.Thread.run(Thread.java:750)

```

## Annotations 定义

@NotEmpty、@NotNull等一系列的校验注解是 Java 的 Bean Validation（JSR）规范， 不过，以上注解本身不做校验。

@Valid 注解来自 Java Bean Validation API (JSR 380)。

@Validated 注解来自 Spring 的定义


@Valid 注解，是 Bean Validation 所定义，可以添加在普通方法、构造方法、方法参数、方法返回、成员变量上，表示它们需要进行约束校验。

```java
package javax.validation;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target({ElementType.METHOD, ElementType.FIELD, ElementType.CONSTRUCTOR, ElementType.PARAMETER, ElementType.TYPE_USE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Valid {
}

```


@Validated ，是 Spring Validation 所定义，可以添加在类、方法参数、普通方法上，表示它们需要进行约束校验。

```java
package org.springframework.validation.annotation;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * Variant of JSR-303's {@link javax.validation.Valid}, supporting the
 * specification of validation groups. Designed for convenient use with
 * Spring's JSR-303 support but not JSR-303 specific.
 *
 * <p>Can be used e.g. with Spring MVC handler methods arguments.
 * Supported through {@link org.springframework.validation.SmartValidator}'s
 * validation hint concept, with validation group classes acting as hint objects.
 *
 * <p>Can also be used with method level validation, indicating that a specific
 * class is supposed to be validated at the method level (acting as a pointcut
 * for the corresponding validation interceptor), but also optionally specifying
 * the validation groups for method-level validation in the annotated class.
 * Applying this annotation at the method level allows for overriding the
 * validation groups for a specific method but does not serve as a pointcut;
 * a class-level annotation is nevertheless necessary to trigger method validation
 * for a specific bean to begin with. Can also be used as a meta-annotation on a
 * custom stereotype annotation or a custom group-specific validated annotation.
 *
 * @author Juergen Hoeller
 * @since 3.1
 * @see javax.validation.Validator#validate(Object, Class[])
 * @see org.springframework.validation.SmartValidator#validate(Object, org.springframework.validation.Errors, Object...)
 * @see org.springframework.validation.beanvalidation.SpringValidatorAdapter
 * @see org.springframework.validation.beanvalidation.MethodValidationPostProcessor
 */
@Target({ElementType.TYPE, ElementType.METHOD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Validated {

	/**
	 * Specify one or more validation groups to apply to the validation step
	 * kicked off by this annotation.
	 * <p>JSR-303 defines validation groups as custom annotations which an application declares
	 * for the sole purpose of using them as type-safe group arguments, as implemented in
	 * {@link org.springframework.validation.beanvalidation.SpringValidatorAdapter}.
	 * <p>Other {@link org.springframework.validation.SmartValidator} implementations may
	 * support class arguments in other ways as well.
	 */
	Class<?>[] value() default {};

}

```




## @Valid 和 @Validated 的区别

### @Valid

@Valid 可以添加在成员变量上，即DTO的成员上，支持嵌套校验。(Nested Objects)

```java
// The @Valid annotation ensures the validation of the whole object.
public class UserAccount {
    
    //...
    
    @Valid
    @NotNull(groups = AdvanceInfo.class)
    private UserAddress useraddress;
}
```

### @Validated

@Validated 注解添加到类上可以启用类级别验证、方法参数验证，并且有 value 属性，支持分组校验，即根据不同的分组采用不同的校验机制。

定义分组
```java
    @NotBlank(message = "Name cannot be blank", groups = {Create.class, Update.class})
    private String name;
```

分组验证
````java
    public ResponseEntity<User> createUser(@Validated(UserDTO.Create.class) @RequestBody UserDTO userDTO) {
        // 创建用户的逻辑
    }
````


## Conclusion

常见的使用方式就是：

启动校验和分组的时候使用 @Validated 注解，

嵌套校验时(Nested Objects)使用 @Valid 注解，

这样就能同时使用分组校验和嵌套校验功能。