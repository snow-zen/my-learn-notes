# DispatcherServlet 工作原理与流程
#Java #Spring 

Spring MVC 中的核心组件就是 DispatcherServlet，它是基于 Java EE 下的 Servlet 进行构建的，它继承于 Java EE 中的 HttpServlet 并将其扩展。

## 初始化

默认情况下，Tomcat 中的 Servlet 是延迟加载的。加载时会回调 init 方法初始化数据，其中主要的步骤是调用 initWebApplicationContext 方法。

**initWebApplicationContext**

当 `webApplicationContext` 为空时，该方法用于初始化 `webApplicationContext`，并配置和刷新上下文。如果是通过 Spring Boot 启动则在创建 DispatcherServlet 时已传入上下文对象，此时无需再初始化。

随后调用 onRefresh 方法对 `webApplicationContext` 进行刷新，而该方法的实现只是个委托方法：

```java
@Override  
protected void onRefresh(ApplicationContext context) {  
   initStrategies(context);  
}
```

它将方法的实现委托给 initStrategies 方法，这个方法主要用于初始化 DispatcherServlet 在调度时所使用到的策略对象，具体方法实现如下：

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

而在这些策略对象中我们只需要关注核心的几个对象即可：MultipartResolver、HandlerMappings、HandlerAdapters、HandlerExceptionResolvers，同时这几个方法的实现也是非常类似：

```java
private void initXxx(ApplicationContext context) {  
    // 从上下文或者 BeanFactory 中获取指定对象，多个则排序
	this.xxx = ...;
	
	if (this.xxx == null) {
		// 如果没找到则获取默认的策略对象
		this.handlerAdapters = getDefaultStrategies(context, Xxx.class);
	}
	
	// 后续初始化处理
}
```

## 处理请求

在 DispatcherServlet 完成初始化过程后即可进行请求的处理，而处理请求的入口方法是 doService 方法，程序在接收到 GET、POST、PUT 和 DELETE 方法时都会回调该方法。

doService 方法内部首先对请求中的以 `org.springframework.web.servlet` 开头的属性进行一个快照，随后向 request 对象中设置一些固定属性，包括 web 上下文对象，最后调用 doDispatch 方法处理请求。

doDispatch 方法是真正处理业务逻辑的核心方法。进入方法后首先检查当前请求是否 Multipart 请求，如果是则通过 MultipartResolver 将请求包装为可处理 Multipart 请求对象：

```java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
	HttpServletRequest processedRequest = request;
	boolean multipartRequestParsed = false;
	// ...
	processedRequest = checkMultipart(request);
	multipartRequestParsed = (processedRequest != request);
	// ...
}
```

得到包装后的请求对象后，通过 `this.handlerMappings` 属性记录的策略对象获取逻辑处理对象以及相关的 HandlerInterceptor 拦截器组合成处理调用链对象：

```java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
	HttpServletRequest processedRequest = request;
	HandlerExecutionChain mappedHandler = null;
	// ...
	mappedHandler = getHandler(processedRequest);  
	if (mappedHandler == null) {  
   		noHandlerFound(processedRequest, response);  
   		return;
	}
	// ...
}

protected HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {  
   if (this.handlerMappings != null) {  
	  // 匹配合适的 HandlerMapping 策略对象组合成 HandlerExecutionChain 处理调用链对象。
      for (HandlerMapping mapping : this.handlerMappings) {  
         HandlerExecutionChain handler = mapping.getHandler(request);  
         if (handler != null) {  
            return handler;  
         }  
      }  
   }  
   return null;  
}

public final HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
	// 获取处理对象
	Object handler = getHandlerInternal(request);
	
	// ...
	
	// 组合成 HandlerExecutionChain 对象
	HandlerExecutionChain executionChain = getHandlerExecutionChain(handler, request);
	
	return executionChain;
}

protected HandlerExecutionChain getHandlerExecutionChain(Object handler, HttpServletRequest request) {  
   HandlerExecutionChain chain = (handler instanceof HandlerExecutionChain ?  
         (HandlerExecutionChain) handler : new HandlerExecutionChain(handler));  
  
   for (HandlerInterceptor interceptor : this.adaptedInterceptors) {  
      if (interceptor instanceof MappedInterceptor) {  
         MappedInterceptor mappedInterceptor = (MappedInterceptor) interceptor;  
         if (mappedInterceptor.matches(request)) {  
            chain.addInterceptor(mappedInterceptor.getInterceptor());  
         }  
      }  
      else {  
         chain.addInterceptor(interceptor);  
      }  
   }  
   return chain;  
}
```

这里有个特殊点：对于普通的 HandlerInterceptor 对象则直接加入调用链。但如果是 MappedInterceptor 子类型的对象则需要通过先调用 matches 方法返回 true 时，才能加入调用链。随后从 `this.handlerAdapters` 属性中匹配一个 HandlerAdapter 对象作为处理适配器：

```java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
	HandlerExecutionChain mappedHandler = null;
	
	// ...
	
	HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());
	
	// ...
}

protected HandlerAdapter getHandlerAdapter(Object handler) throws ServletException {  
   if (this.handlerAdapters != null) {  
      for (HandlerAdapter adapter : this.handlerAdapters) {  
         if (adapter.supports(handler)) {  
            return adapter;  
         }  
      }  
   }  
   throw new ServletException("No adapter for handler [" + handler +  
         "]: The DispatcherServlet configuration needs to include a HandlerAdapter that supports this handler");  
}
```

在开始处理业务逻辑前，还有一处拦截器的前置调用：

```java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
	HandlerExecutionChain mappedHandler = null;
	
	// ...
	
	if (!mappedHandler.applyPreHandle(processedRequest, response)) {  
   		return;  
	}
	
	// ...
}

boolean applyPreHandle(HttpServletRequest request, HttpServletResponse response) throws Exception {  
   HandlerInterceptor[] interceptors = getInterceptors();  
   if (!ObjectUtils.isEmpty(interceptors)) {  
      for (int i = 0; i < interceptors.length; i++) {  
         HandlerInterceptor interceptor = interceptors[i];  
         if (!interceptor.preHandle(request, response, this.handler)) {  
            triggerAfterCompletion(request, response, null);  
            return false;         }  
         this.interceptorIndex = i;  
      }  
   }  
   return true;  
}
```

前置调用完成后，真正的业务逻辑处理便开始了：

```java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
	HttpServletRequest processedRequest = request;
	HandlerExecutionChain mappedHandler = null;
	ModelAndView mv = null;
	HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());
	
	// ...
	
	mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
	
	// ...
}
```

业务逻辑处理完成后，开始回调 HandlerInterceptor 拦截器的 postHandle 方法：

```java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
	HttpServletRequest processedRequest = request;
	HandlerExecutionChain mappedHandler = null;
	ModelAndView mv = null;
	
	// ...
	
	mappedHandler.applyPostHandle(processedRequest, response, mv);
	
	// ...
}
```

最后处理调度结果：

```java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
	HttpServletRequest processedRequest = request;
	HandlerExecutionChain mappedHandler = null;
	
	Exception dispatchException = null;
	try {
		// ...
	} catch (Exception ex) {
		dispatchException = ex;
	}
	processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
	
	// ...
}

private void processDispatchResult(HttpServletRequest request, HttpServletResponse response,  
      @Nullable HandlerExecutionChain mappedHandler, @Nullable ModelAndView mv,  
      @Nullable Exception exception) throws Exception {
	
	// 如果有异常出现
	if (exception != null) {  
	   if (exception instanceof ModelAndViewDefiningException) {  
		  logger.debug("ModelAndViewDefiningException encountered", exception);  
		  mv = ((ModelAndViewDefiningException) exception).getModelAndView();  
	   }  
	   else {  
		  Object handler = (mappedHandler != null ? mappedHandler.getHandler() : null);  
		  mv = processHandlerException(request, response, handler, exception);  
		  errorView = (mv != null);  
	   }  
	}
	
	// 渲染视图
	if (mv != null && !mv.wasCleared()) {  
	   render(mv, request, response);  
	   if (errorView) {  
		  WebUtils.clearErrorRequestAttributes(request);  
	   }  
	}  
	else {  
	   if (logger.isTraceEnabled()) {  
		  logger.trace("No view rendering, null ModelAndView returned.");  
	   }  
	}
	
	if (mappedHandler != null) {  
	   // 回调
	   mappedHandler.triggerAfterCompletion(request, response, null); 
	}
}
```

处理回调结果时，主要进行了三件事：

+ 如果有异常出现，则回调 HandlerExceptionResolver 中的 resolveException 方法。
+ 如果 ModelAndView 对象不为 null 时，渲染视图。
+ 通过 HandlerExecutionChain 对象回调所有 HandlerInterceptor 拦截器的 afterCompletion 方法。

## 解析参数与返回值

Spring MVC 中的参数由 HandlerMethodArgumentResolver 对象进行解析，在 RequestMappingHandlerAdapter 方法的 afterPropertiesSet 回调方法中添加默认的参数解析器。

参数解析器的定义如下：

```java
public interface HandlerMethodArgumentResolver {  
    boolean supportsParameter(MethodParameter parameter);  
  
    @Nullable  
    Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer, NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception;  
}
```

如果需要自定义参数类型解析器，可以让 `@Configuratio`n 配置类实现 WebMvcConfigurer 接口并重写 addArgumentResolvers 方法即可。

对于返回值则是由 HandlerMethodReturnValueHandler 对象进行解析，同样也是在 RequestMappingHandlerAdapter 方法的 afterPropertiesSet 回调方法中添加默认的参数解析器。

返回值解析器的定义如下：

```java
public interface HandlerMethodReturnValueHandler {  
    boolean supportsReturnType(MethodParameter returnType);  
  
    void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType, ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception;  
}
```

另外，如果需要自定义返回值解析器，也是通过让 `@Configuratio`n 配置类实现 WebMvcConfigurer 接口并重写 addReturnValueHandlers 方法即可。

## 异常处理

异常处理是在最后处理调度结果时进行的。当业务处理方法有抛出异常时，则会从 this.handlerExceptionResolvers 属性中获取一个 HandlerExceptionResolver 对象解析异常：

```java
protected ModelAndView processHandlerException(HttpServletRequest request, HttpServletResponse response,  
      @Nullable Object handler, Exception ex) throws Exception {
	ModelAndView exMv = null;
	if (this.handlerExceptionResolvers != null) {  
	   for (HandlerExceptionResolver resolver : this.handlerExceptionResolvers) {  
		  exMv = resolver.resolveException(request, response, handler, ex);  
		  if (exMv != null) {  
			 break;  
		  }  
	   }  
	}
	
	// ...
}
```

Tip：使用 `@ExceptionHandler` 标注的方法也是在此处回调的，如果是返回 JSON 格式数据则默认会返回一个具有空值的 ModelAndView 对象。

