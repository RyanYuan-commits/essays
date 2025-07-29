## DispatcherServlet 主流程

### 初始化流程

也就是 Servlet 生命周期中的 init 方法。

- org.springframework.web.servlet.**HttpServletBean**#init()：DispatcherServlet 初始化的起点
- org.springframework.web.servlet.**FrameworkServlet**#initServletBean：调用方法 initWebApplicationContext 来初始化 Web 的 ApplicationContext，这是对 Spring 原生 ApplicationContext 的封装
- org.springframework.web.servlet.**FrameworkServlet**#initWebApplicationContext
- org.springframework.web.servlet.**FrameworkServlet**#configureAndRefreshWebApplicationContext：其中会调用 Spring 中的 refresh 方法来初始化容器。
- org.springframework.web.servlet.**DispatcherServlet**#onRefresh
- org.springframework.web.servlet.**DispatcherServlet**#initStrategies：初始化 SpringMVC 运行过程中需要的各种核心类

>[!info] org.springframework.web.servlet.DispatcherServlet#initStrategies
```java
protected void initStrategies(ApplicationContext context) {  
    initMultipartResolver(context);  
    initLocaleResolver(context);  
    initThemeResolver(context);  
    initHandlerMappings(context);  
    initHandlerAdapters(context);  
    initHandlerExceptionResolvers(context);  
    initRequestToViewNameTranslator(context);  
    initViewResolvers(context);  
    initFlashMapManager(context);  
}
```

在 DispatcherServlet 的静态代码块中，会加载默认的一些资源（比如 HandlerMapping、HandlerAdapter 等）：
```java
private static final String DEFAULT_STRATEGIES_PATH = "DispatcherServlet.properties";

static {  
    // Load default strategy implementations from properties file.  
    // This is currently strictly internal and not meant to be customized    
    // by application developers.    
    try {  
       ClassPathResource resource = new ClassPathResource(DEFAULT_STRATEGIES_PATH, DispatcherServlet.class);  
       defaultStrategies = PropertiesLoaderUtils.loadProperties(resource);  
    }  
    catch (IOException ex) {  
       throw new IllegalStateException("Could not load 'DispatcherServlet.properties': " + ex.getMessage());  
    }  
}
```

![[DispatcherServlet.properties 配置文件.png]]

配置文件中配置了一些关键类的默认实现，如果开发者没有向 Spring 容器中注入自己的实现的话，SpringMVC 会自动加载上述的实现。

### 执行流程

对应的是 Servlet 生命周期中的 service 方法。

- javax.servlet.http.**HttpServlet**#service：调用本类中的 doGet 和 doPost
- org.springframework.web.servlet.FrameworkServlet#processRequest：FrameworkServlet 中重写了 doGet 和 doPost 方法，其最终都会调用到 processRequest 方法，方法中调用了 doService 方法

>[!info] FrameworkServlet 中对 doGet 和 doPost 方法
```java
@Override  
protected final void doGet(HttpServletRequest request, HttpServletResponse response)  
       throws ServletException, IOException {  
    processRequest(request, response);  
}  
@Override  
protected final void doPost(HttpServletRequest request, HttpServletResponse response)  
       throws ServletException, IOException {  
    processRequest(request, response);  
}
```
- org.springframework.web.servlet.**DispatcherServlet**#doService：调用 doDispatch 核心方法
- org.springframework.web.servlet.**DispatcherServlet**#doDispatch

>[!info] org.springframework.web.servlet.DispatcherServlet#doDispatch
```java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {  
    HttpServletRequest processedRequest = request;  
    HandlerExecutionChain mappedHandler = null;  
    boolean multipartRequestParsed = false;  
  
    WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);  
  
    try {  
       ModelAndView mv = null;  
       Exception dispatchException = null;  
  
       try {  
	       // 判断这个请求是否包含多部件（文件）请求
          processedRequest = checkMultipart(request);  
          multipartRequestParsed = (processedRequest != request);  
  
          // Determine handler for the current request.
          // 根据请求查找 Handler，类型是 HandlerExecutionChain
          mappedHandler = getHandler(processedRequest);  
          if (mappedHandler == null || mappedHandler.getHandler() == null) {  
             noHandlerFound(processedRequest, response);  
             return;  
          }  
  
          // Determine handler adapter for the current request.  
          HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());  
  
          // Process last-modified header, if supported by the handler.  
          String method = request.getMethod();  
          boolean isGet = "GET".equals(method);  
          if (isGet || "HEAD".equals(method)) {  
             long lastModified = ha.getLastModified(request, mappedHandler.getHandler());  
             if (logger.isDebugEnabled()) {  
                logger.debug("Last-Modified value for [" + getRequestUri(request) + "] is: " + lastModified);  
             }  
             if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {  
                return;  
             }  
          }  
  
          if (!mappedHandler.applyPreHandle(processedRequest, response)) {  
             return;  
          }  
  
          // Actually invoke the handler.  
          mv = ha.handle(processedRequest, response, mappedHandler.getHandler());  
  
          if (asyncManager.isConcurrentHandlingStarted()) {  
             return;  
          }  
  
          applyDefaultViewName(processedRequest, mv);  
          mappedHandler.applyPostHandle(processedRequest, response, mv);  
       }  
       catch (Exception ex) {  
          dispatchException = ex;  
       }  
       processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);  
    }  
    catch (Exception ex) {  
       triggerAfterCompletion(processedRequest, response, mappedHandler, ex);  
    }  
    catch (Error err) {  
       triggerAfterCompletionWithError(processedRequest, response, mappedHandler, err);  
    }  
    finally {  
       if (asyncManager.isConcurrentHandlingStarted()) {  
          // Instead of postHandle and afterCompletion  
          if (mappedHandler != null) {  
             mappedHandler.applyAfterConcurrentHandlingStarted(processedRequest, response);  
          }  
       }  
       else {  
          // Clean up any resources used by a multipart request.  
          if (multipartRequestParsed) {  
             cleanupMultipart(processedRequest);  
          }  
       }  
    }  
}
```
