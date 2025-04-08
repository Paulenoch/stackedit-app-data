# IOC，控制反转
IoC 的思想就是将原本在程序中手动创建对象的控制权，交由 Spring 框架来管理
-   **控制**：指的是对象创建（实例化、管理）的权力
-   **反转**：控制权交给外部环境（Spring 框架、IoC 容器）

将对象之间的相互依赖关系交给 IoC 容器来管理，并由 IoC 容器完成对象的注入。这样可以很大程度上简化应用的开发，把应用从复杂的依赖关系中解放出来。 IoC 容器就像是一个工厂一样，当我们需要创建一个对象的时候，只需要配置好配置文件/注解即可，完全不用考虑对象是如何被创建出来的。

在 Spring 中， IoC 容器是 Spring 用来实现 IoC 的载体， IoC 容器实际上就是个 Map（key，value），Map 中存放的是各种对象。

# Spring Bean

Bean 代指的就是那些被 IoC 容器所管理的对象。

### Bean
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTEyMjQ3OTE2NzQsMTcxMjc1NTk5MV19
-->