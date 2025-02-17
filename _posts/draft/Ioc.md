设计模式

applicationEvent 就是一个观察者模式的扩展 基于Java的标准事件
public abstract class ApplicationEvent extends EventObject {

JdbcTemplate 是一个模板方法模式的实现  模版方法 反转控制

前缀模式

Enable 模式
Configuration 模式

后缀模式

处理器模式 
Processor
Resolver
Handler

意识模式
Aware

配置器模式
Configuror

选择器模式
Selector

The org.springframework.beans and org.springframework.context packages are the basis for Spring Framework’s IoC container. The BeanFactory interface provides an advanced configuration mechanism capable of managing any type of object. ApplicationContext is a sub-interface of BeanFactory. It adds:

Easier integration with Spring’s AOP features

Message resource handling (for use in internationalization)

Event publication

Application-layer specific contexts such as the WebApplicationContext for use in web applications.

beanFactory 有的 applicationContext 都有


org.springframework.context.ConfigurableApplicationContext#refresh

org.springframework.context.support.AbstractApplicationContext#refresh

org.springframework.context.support.AbstractApplicationContext#close

bean 属性 和 说明

Class Bean全类名，必须是具体类
Name  Bean的名称或者ID
Scope  Bean的作用域 singleton prototype
Constructor arguments Bean 的构造器参数 用于依赖注入
Properties Bean的属性设置 用于依赖注入
Autowiring mode  Bean的自动绑定模式
Lazy initialization mode  Bean的延迟初始化模式（延迟和非延迟)
Initialization method Bean初始化回调方法名称
Destruction method Bean销毁回调方法名称

bean的命名规则
org.springframework.beans.factory.support.BeanNameGenerator#generateBeanName


bean 注册
@Bean
@Component
@Import





Using a FactoryBean can be a good practice to encapsulate complex 
construction logic or make configuring highly configurable objects easier in Spring.
https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/beans/factory/FactoryBean.html
org.springframework.beans.factory.FactoryBean
org.springframework.beans.factory.BeanFactory











https://docs.spring.io/spring-framework/reference/core/beans/introduction.html