IoC
inversion of control

从Hollywood Principle发展而来

Martin Fowler inversion of control containers and the dependency injection pattern

Martin Fowler InversionOfControl



IoC can be implemented in different ways, there are two main types:
Dependency Lookup
Dependency Injection

Inversion of control is sometimes facetiously referred to as the "Hollywood Principle: Don't call us, we'll call you."

google guice 开源的ioc容器




构造器注入 依赖的对象不变 完整的初始化状态 避免状态被修改  参数过多就是坏味道(bad code smell)

applicationContext 上下文


Spring 内建的依赖 Bean
Spring 容器会自动注册一些 核心组件，这些组件可以直接在我们的应用中使用，例如：

内建 Bean	作用
ApplicationContext	Spring 容器上下文，管理所有 Bean
BeanFactory	Spring Bean 工厂，用于 Bean 的管理
Environment	环境变量，用于获取配置属性（如 application.properties）
ResourceLoader	资源加载器，用于加载文件、类路径资源
ApplicationEventPublisher	事件发布器，用于发布 Spring 事件
MessageSource	国际化消息解析，用于 i18n


