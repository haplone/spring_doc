# Spring MVC源码解析--HandlerMapping接口实现类职责分析

AbstractHandlerMapping
HandlerMapping接口的抽象实现。支持排序，包含默认处理器，拦截器，包括基于url映射的拦截器。

AbstactHandlerMethodMapping
定义request与HandlerMethod直接的映射。每一个处理器，都有子类维护着一个唯一的映射<T>,这边还只是一个泛型。

RequestMappingInfoHandlerMapping
将request与Handler直接的映射泛型具化为RequestMappingInfo

RequestMappingHandlerMapping
查找使用@Controller注解的类中，使用@RequestMapping的方法，并生成RequestMappingInfo实例

AbstractUrlHandlerMapping
实现HandlerMapping接口，基于url映射的抽象基类。提供url到控制器的映射。url的匹配直接直接匹配也支持ant方式匹配。


SimpleUrlHandlerMapping
通过mapping属性设置url到控制器的映射。这边可以映射到类实例，spring容器中的bean name（需要非单例)

AbstractDetectingUrlHandlerMapping
通过扫描spring容器中的类来url到控制的映射关系。

BeanNameUrlHandlerMapping
定义url的生成规则时使用"/"开头的bean name或alias。

AbstractControllerUrlHandlerMapping
细化处理器过滤规则，要求不在排除的package内，且是Controller的子类

ControllerBeanNameHandlerMapping
明确url的生成规则：使用bean name。这边的bean name不需要“/”开头,会自动补全。

ControllerClassNameHandlerMapping
明确url的生成规则：使用class name。
如果是Controller的子类，映射规则是这样的:
* WelcomeController -> /welcome*
* HomeController -> /home*

如果时MultiActionControllers子类并有@Controller注解，映射规则是这样的：
* WelcomeController -> /welcome, /welcome/*
* CatalogController -> /catalog, /catalog/*
