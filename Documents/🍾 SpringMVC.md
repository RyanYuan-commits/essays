![[Spring MVC 执行流程图.png]]
# 前置知识储备
## Servlet 的生命周期方法
- init：Servlet 对象创建之后调用
- service：Servlet 对象被 HTTP 请求访问时调用
- destroy：Servlet 对象销毁之前调用
## DispatcherServlet 继承体系
![[DispatcherServlet 继承体系.png]]
[[🍊【SpringMVC】Dispatcher 执行流程]]
[[♨️【SpringMVC】RequestMappingHandlerMapping 流程]]
[[🏙️【SpringMVC】RequestMappingHandlerAdapter 处理流程]]