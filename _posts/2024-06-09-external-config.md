---
layout: post
title: "Spring Boot Externalized Configuration"
date: 2024-06-09 10:00:00 +0800
tags: [ Jekyll, Markdown, 博客 ]
author: "孙珂"
---

#  Spring Boot Externalized Configuration

## 相关源码

根据源码，可以得到相关配置的加载顺序。

```java
	/**
     * org.springframework.boot.context.config.ConfigDataEnvironment
     *
	 * Default search locations used if not {@link #LOCATION_PROPERTY} is found.
	 */
	static final ConfigDataLocation[] DEFAULT_SEARCH_LOCATIONS;
	static {
		List<ConfigDataLocation> locations = new ArrayList<>();
		locations.add(ConfigDataLocation.of("optional:classpath:/;optional:classpath:/config/"));
		locations.add(ConfigDataLocation.of("optional:file:./;optional:file:./config/;optional:file:./config/*/"));
		DEFAULT_SEARCH_LOCATIONS = locations.toArray(new ConfigDataLocation[0]);
	}
```

## 配置信息加载顺序

当需要加载 my.config.version 这个配置信息的时候，Spring Boot 是如何根据目录的优先级找到对应的配置信息的呢?

```java
   @Value("${my.config.version}")
   private String myConfigVersion;
```

当前的项目目录为 /codeComplete/think-in-spring/

优先级最低的是 classpath 根目录的 application.properties 文件

目录地址: /codeComplete/think-in-spring/src/main/resources/application.properties

```
my.config.version=v1/codeComplete/think-in-spring/src/main/resources/application.properties
```

优先级第二  classpath 目录的下面 config 目录下面的 application.properties 文件

目录地址: /codeComplete/think-in-spring/src/main/resources/config/application.properties

```
my.config.version=v2/codeComplete/think-in-spring/src/main/resources/config/application.properties
```


优先级第三 是当前项目的目录下面的 application.properties 文件

目录地址: /codeComplete/application.properties

```
my.config.version=v3/codeComplete/application.properties
```

优先级第四 是当前项目的目录 config 下面 application.properties 文件

目录地址: /codeComplete/config/application.properties

```
my.config.version=v4/codeComplete/config/application.properties
```


优先级第五 是当前项目的目录 config 下面的子目录下面的 application.properties 文件

目录地址: /codeComplete/config/dev/application.properties

```
my.config.version=v5/codeComplete/config/dev/application.properties
```




## 相关文档

>External Application Properties
>Spring Boot will automatically find and load application.properties and application.yaml files from the following locations when your application starts:
>
>1.From the classpath
>
>    a.The classpath root
>
>    b.The classpath /config package
>
>2.From the current directory
>
>    a.The current directory
>
>    b.The config/ subdirectory in the current directory
>
>    c.Immediate child directories of the config/ subdirectory
>
>The list is ordered by precedence (with values from lower items overriding earlier ones). Documents from the loaded files are added as PropertySource instances to the Spring Environment.



[spring boot 官方文档](https://docs.spring.io/spring-boot/reference/features/external-config.html#features.external-config)。
