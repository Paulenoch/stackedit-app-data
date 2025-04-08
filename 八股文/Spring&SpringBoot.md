# IOC，控制反转
IoC 的思想就是将原本在程序中手动创建对象的控制权，交由 Spring 框架来管理
-   **控制**：指的是对象创建（实例化、管理）的权力
-   **反转**：控制权交给外部环境（Spring 框架、IoC 容器）

将对象之间的相互依赖关系交给 IoC 容器来管理，并由 IoC 容器完成对象的注入。这样可以很大程度上简化应用的开发，把应用从复杂的依赖关系中解放出来。 IoC 容器就像是一个工厂一样，当我们需要创建一个对象的时候，只需要配置好配置文件/注解即可，完全不用考虑对象是如何被创建出来的。

在 Spring 中， IoC 容器是 Spring 用来实现 IoC 的载体， IoC 容器实际上就是个 Map（key，value），Map 中存放的是各种对象。

# Spring Bean

Bean 代指的就是那些被 IoC 容器所管理的对象。

### Bean的生命周期
实例化——属性赋值——初始化——销毁
![输入图片说明](/imgs/2025-04-08/6LcooDzFRf5DpS3l.png)

### 注入Bean的方式
1. 构造函数注入：通过类的构造函数注入
2. Setter注入：通过类的Setter方法注入
3. Field（字段）注入，直接在类的字段上使用注解（@Autowired，@Resource）注入

推荐构造函数注入：依赖完整性，不可变，初始化保证，测试便利

# Spring AOP
AOP 的目的是将横切关注点（如日志记录、事务管理、权限控制、接口限流、接口幂等等）从核心业务逻辑中分离出来，通过动态代理、字节码操作等技术，实现代码的复用和解耦，提高代码的可维护性和可扩展性。OOP 的目的是将业务逻辑按照对象的属性和行为进行封装，通过类、对象、继承、多态等概念，实现代码的模块化和层次化（也能实现代码的复用），提高代码的可读性和可维护性。

# SpringBoot自动装配

没有 Spring Boot 的情况下，如果我们需要引入第三方依赖，需要手动配置，非常麻烦。但是，Spring Boot 中，我们直接引入一个 starter 即可。比如你想要在项目中使用 redis 的话，直接在项目中引入对应的 starter 即可。

引入 starter 之后，我们通过少量注解和一些简单的配置就能使用第三方组件提供的功能了。

在我看来，自动装配可以简单理解为：**通过注解或者一些简单的配置就能在 Spring Boot 的帮助下实现某块功能。**

SpringBoot的核心注解`@SpringBootApplication`
由`@Configuration`、`@EnableAutoConfiguration`、`@ComponentScan` 组成
-   `@EnableAutoConfiguration`：启用 SpringBoot 的自动配置机制
-   `@Configuration`：允许在上下文中注册额外的 bean 或导入其他配置类
-   `@ComponentScan`：扫描被`@Component` (`@Service`,`@Controller`)注解的 bean，注解默认会扫描启动类所在的包下所有的类 ，可以自定义不扫描某些 bean。如下图所示，容器中将排除`TypeExcludeFilter`和`AutoConfigurationExcludeFilter`。

<!--stackedit_data:
eyJoaXN0b3J5IjpbLTg4NjcxNjcyNyw1MDAxNzQ0MjUsMTcxMj
c1NTk5MV19
-->