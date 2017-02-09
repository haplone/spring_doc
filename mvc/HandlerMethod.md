# Spring MVC源码解析--HandlerMethod及其子类职责分析

HandlerMethod
封装了处理器相关的类和方法信息。提供对于方法参数、方法返回值、方法注解的便捷访问。
提供createWithResolvedBean方法，可以去spring容器中通过bean name查找实例。
[使用场景]: HandlerMapping封装处理器用。

InvocableHandlerMethod
提供一个可以触发处理器的方法。使用HandlerMethodArgumentResolvers解决方法参数准备问题。
方法参数的解决需要使用WebDataBinder来进行数据绑定或类型转化。
[使用场景]: 执行使用@ModelAttribute注解时，执行方法。


ServletInvocableHandlerMethod
添加返回值处理功能。
使用HandlerMethodReturnValueHandler处理返回值，支持使用@ResponseStatus设置response status。
[使用场景]: HandlerAdapter执行处理器时使用。？?

