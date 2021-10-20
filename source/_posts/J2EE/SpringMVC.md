---
title: SpringMVC源码浅析
date: 2017-12-20
categories: J2EE
---


先来看一张整体的处理过程图：

![SpringMVC原理图](http://img-blog.csdn.net/20171225214823279?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

> 下面的Spring源码版本为4.3.12，是目前最新的稳定版本。
>
> 源码版本不一致，可能会有稍许差异。

# 请求的分发

***DispatcherServlet.doDispatcher()***

先看一下DispatcherServlet方法的核心方法doDispatch：

```java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
	HttpServletRequest processedRequest = request;
	HandlerExecutionChain mappedHandler = null;
	boolean multipartRequestParsed = false;

    // 这个主要用于处理异步任务
    // handler可以返回Callable、WebAsyncTask、ListenableFuture等对象
    // 这些任务都会交给WebAsyncManager处理，不过这些特性需要Servlet3.0容器的支持
	WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);

	try {
		ModelAndView mv = null;
		Exception dispatchException = null;

		try {
            // 检查是否为multipart/form-data请求(文件上传)，
            // 如果是则使用multipartResolver解析请求。
			processedRequest = checkMultipart(request);
			multipartRequestParsed = (processedRequest != request);

			// 根据请求匹配handlerMappings
            // 调用HandlerMapping.getHandler方法得到HandlerExecutionChain
            // HandlerExecutionChain中包装了拦截器链和handler
			mappedHandler = getHandler(processedRequest);
			if (mappedHandler == null || mappedHandler.getHandler() == null) {
                // 没找到handler就返回
				noHandlerFound(processedRequest, response);
				return;
			}

			// 根据handler的类型选用合适的HandlerAdapter
            // HandlerAdapter用来适配不同类型的handler
			HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());


			String method = request.getMethod();
			boolean isGet = "GET".equals(method);
			if (isGet || "HEAD".equals(method)) {
                // 如果handler支持last-modified头信息，则处理该头信息
                // 查看handler中指定的资源过期时间
				long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
				if (logger.isDebugEnabled()) {
					logger.debug("Last-Modified value for [" + getRequestUri(request) + "] is: " + lastModified);
				}
				if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
					return;
				}
			}

            // 调用过滤器链的preHandle方法
            // 如果拦截器返回false，则直接返回
			if (!mappedHandler.applyPreHandle(processedRequest, response)) {
				return;
			}

			// 真正调用Handler的地方
			mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

			if (asyncManager.isConcurrentHandlingStarted()) {
				return;
			}

            // 如果没有返回view则尝试使用默认的view
			applyDefaultViewName(processedRequest, mv);

            // 调用过滤器链的postHandle方法
            // postHandle方法中可以操作ModelAndView
			mappedHandler.applyPostHandle(processedRequest, response, mv);
		}
		catch (Exception ex) {
			dispatchException = ex;
		}
		catch (Throwable err) {
			// As of 4.3, we're processing Errors thrown from handler methods as well,
			// making them available for @ExceptionHandler methods and other scenarios.
			dispatchException = new NestedServletException("Handler dispatch failed", err);
		}
        // 解析View
        // 渲染ModelAndView
        // 调用过滤器链的afterCompletion方法
		processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
	}
	catch (Exception ex) {
        // 调用过滤器链的afterCompletion方法
		triggerAfterCompletion(processedRequest, response, mappedHandler, ex);
	}
	catch (Throwable err) {
        // 调用过滤器链的afterCompletion方法
		triggerAfterCompletion(processedRequest, response, mappedHandler,
				new NestedServletException("Handler processing failed", err));
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

# 请求处理器的执行链

***HandlerExecutionChain***

HandlerExecutionChain主要负责拦截器链的调用，另外还保存了handler的引用。

```java
public class HandlerExecutionChain {

	private final Object handler;

	private HandlerInterceptor[] interceptors;

	private List<HandlerInterceptor> interceptorList;

	private int interceptorIndex = -1;

	// 执行拦截器链的preHandle方法
	boolean applyPreHandle(HttpServletRequest request, HttpServletResponse response) throws Exception {
		HandlerInterceptor[] interceptors = getInterceptors();
		if (!ObjectUtils.isEmpty(interceptors)) {
            // 正序
			for (int i = 0; i < interceptors.length; i++) {
				HandlerInterceptor interceptor = interceptors[i];
                // 执行拦截器链的preHandle方法
				if (!interceptor.preHandle(request, response, this.handler)) {
                    // 如果拦截器返回false，则直接调用triggerAfterCompletion，然后并返回
					triggerAfterCompletion(request, response, null);
					return false;
				}
				this.interceptorIndex = i;
			}
		}
		return true;
	}

    // 执行拦截器链的postHandle方法
	void applyPostHandle(HttpServletRequest request, HttpServletResponse response, ModelAndView mv) throws Exception {
		HandlerInterceptor[] interceptors = getInterceptors();
		if (!ObjectUtils.isEmpty(interceptors)) {
            // 逆序
			for (int i = interceptors.length - 1; i >= 0; i--) {
				HandlerInterceptor interceptor = interceptors[i];
				interceptor.postHandle(request, response, this.handler, mv);
			}
		}
	}

    // 执行拦截器链的afterCompletion方法
	void triggerAfterCompletion(HttpServletRequest request, HttpServletResponse response, Exception ex)
			throws Exception {

		HandlerInterceptor[] interceptors = getInterceptors();
		if (!ObjectUtils.isEmpty(interceptors)) {
            // 逆序
			for (int i = this.interceptorIndex; i >= 0; i--) {
				HandlerInterceptor interceptor = interceptors[i];
				try {
					interceptor.afterCompletion(request, response, this.handler, ex);
				}
				catch (Throwable ex2) {
					logger.error("HandlerInterceptor.afterCompletion threw exception", ex2);
				}
			}
		}
	}
    ...
}
```

从上面的源码可以大致得到拦截器链与handler执行的先后顺序：

![过滤链执行顺序](http://img-blog.csdn.net/20171226164722543?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

# 检查并解析文件上传请求

***DispatcherServlet.checkMultipart()***

这个部分在图上没有体现，因为文件上传的请求在项目中占的比重肯定不会很多：

```java
protected HttpServletRequest checkMultipart(HttpServletRequest request) throws MultipartException {
	if (this.multipartResolver != null && this.multipartResolver.isMultipart(request)) {
		if (WebUtils.getNativeRequest(request, MultipartHttpServletRequest.class) != null) {
            // 请求已经被解析
			logger.debug("Request is already a MultipartHttpServletRequest - if not in a forward, " +
					"this typically results from an additional MultipartFilter in web.xml");
		}
		else if (hasMultipartException(request) ) {
            // 请求已经解析失败了
			logger.debug("Multipart resolution failed for current request before - " +
					"skipping re-resolution for undisturbed error rendering");
		}
		else {
			try {
                // 使用multipartResolver去解析请求
                // Spring提供了对Apache Commons-Fileupload的支持类：CommonsMultipartResolver
                // 另外Spring还提供了Servlet3.0标准中的文件上传的支持类：StandardServletMultipartResolver
				return this.multipartResolver.resolveMultipart(request);
			}
			catch (MultipartException ex) {
				if (request.getAttribute(WebUtils.ERROR_EXCEPTION_ATTRIBUTE) != null) {
					logger.debug("Multipart resolution failed for error dispatch", ex);
					// Keep processing error dispatch with regular request handle below
				}
				else {
					throw ex;
				}
			}
		}
	}
	// If not returned before: return original request.
	return request;
}
```

# 获取请求处理器

***DispacherServlet.getHandler()***

```java
protected HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
    // 遍历所有的HandlerMapping
	for (HandlerMapping hm : this.handlerMappings) {
		if (logger.isTraceEnabled()) {
			logger.trace(
					"Testing handler map [" + hm + "] in DispatcherServlet with name '" + getServletName() + "'");
		}
        // 根据请求uri得到处理请求的handler
        // 根据请求uri得到所有匹配的interceptor
        // 将handler和interceptor包装在一个HandlerExecutionChain中
		HandlerExecutionChain handler = hm.getHandler(request);
		if (handler != null) {
			return handler;
		}
	}
	return null;
}
```

![HandlerMapping](http://img-blog.csdn.net/20171225214959386?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

## RequestMappingHandlerMapping

RequestMappingHandlerMapping是匹配@Controller和@RequestMapping注解的，这种方式也是最常用的：

```java
public class RequestMappingHandlerMapping extends RequestMappingInfoHandlerMapping
        implements MatchableHandlerMapping, EmbeddedValueResolverAware {
    ...
	@Override
	protected boolean isHandler(Class<?> beanType) {
        // RequestMappingHandlerMapping支持@Controller注解或RequestMapping注解
		return (AnnotatedElementUtils.hasAnnotation(beanType, Controller.class) ||
				AnnotatedElementUtils.hasAnnotation(beanType, RequestMapping.class));
	}

    /**
     * RequestMappingHandlerMapping中getHandler来自父类AbstractHandlerMapping
     * 为了减少篇幅我直接把父类的方法放在这里。
     */
	public final HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
        // 根据请求uri得到处理请求的handler
		Object handler = getHandlerInternal(request);
		if (handler == null) {
			handler = getDefaultHandler();
		}
		if (handler == null) {
			return null;
		}
		// Bean name or resolved handler?
		if (handler instanceof String) {
			String handlerName = (String) handler;
			handler = getApplicationContext().getBean(handlerName);
		}

        // 根据请求uri得到所有匹配的interceptor
        // 将handler和interceptor包装在一个HandlerExecutionChain中
		HandlerExecutionChain executionChain = getHandlerExecutionChain(handler, request);
		if (CorsUtils.isCorsRequest(request)) {
			CorsConfiguration globalConfig = this.globalCorsConfigSource.getCorsConfiguration(request);
			CorsConfiguration handlerConfig = getCorsConfiguration(handler, request);
			CorsConfiguration config = (globalConfig != null ? globalConfig.combine(handlerConfig) : handlerConfig);
			executionChain = getCorsHandlerExecutionChain(request, executionChain, config);
		}
		return executionChain;
	}
    /**
     * RequestMappingHandlerMapping中getHandlerInternal来自父类AbstractHandlerMethodMapping
     * 为了减少篇幅我直接把父类的方法放在这里。
     */
	protected HandlerMethod getHandlerInternal(HttpServletRequest request) throws Exception {
		String lookupPath = getUrlPathHelper().getLookupPathForRequest(request);
		if (logger.isDebugEnabled()) {
			logger.debug("Looking up handler method for path " + lookupPath);
		}
		this.mappingRegistry.acquireReadLock();
		try {
            /**
             *  // HandlerMethod代表处理请求的方法
             *	public class HandlerMethod {
             *      // 加了@Controller注解的bean
             *      private final Object bean;
             *      private final BeanFactory beanFactory;
             *      private final Class<?> beanType;
             *      // 加了@RequestMapping注解的方法
             *      private final Method method;
             *      private final Method bridgedMethod;
             *      // 方法参数，参数上可能加了@PathVariable,@RequestParam,
             *      // @RequestHeader,@CookieValue等注解
             *      private final MethodParameter[] parameters;
             *      // 方法上可能有ResponseStatus注解
             *      private HttpStatus responseStatus;
             *      private String responseStatusReason;
             *      private HandlerMethod resolvedFromHandlerMethod;
             *	}
             *
             */
            // 处理请求的方法
			HandlerMethod handlerMethod = lookupHandlerMethod(lookupPath, request);
			if (logger.isDebugEnabled()) {
				if (handlerMethod != null) {
					logger.debug("Returning handler method [" + handlerMethod + "]");
				}
				else {
					logger.debug("Did not find handler method for [" + lookupPath + "]");
				}
			}
			return (handlerMethod != null ? handlerMethod.createWithResolvedBean() : null);
		}
		finally {
			this.mappingRegistry.releaseReadLock();
		}
	}
}
```

# 获取处理器适配器

***DispatcherServlet.getHandlerAdapter()***

```java
protected HandlerAdapter getHandlerAdapter(Object handler) throws ServletException {
    // 遍历所有的HandlerAdapter
	for (HandlerAdapter ha : this.handlerAdapters) {
		if (logger.isTraceEnabled()) {
			logger.trace("Testing handler adapter [" + ha + "]");
		}
        // 找到一个支持处理该类型handler的HandlerAdapter并返回
		if (ha.supports(handler)) {
			return ha;
		}
	}
    // 没有合适的HandlerAdapter将会抛异常
	throw new ServletException("No adapter for handler [" + handler +
			"]: The DispatcherServlet configuration needs to include a HandlerAdapter that supports this handler");
}
```

![HandlerAdapter](http://img-blog.csdn.net/20171225215022458?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

以上的HandlerAdapter分别用来适配以下几种控制器：

* SimpleServletHandlerAdapter适配`javax.servlet.Servlet`接口
* SimpleControllerHandlerAdapter适配`org.springframework.web.servlet.mvc.Controller`接口(这是Spring2.5之前的控制器需要实现的接口)
* HttpRequestHandlerAdapter适配`org.springframework.web.HttpRequestHandler`接口，包括ResourceHttpRequestHandler处理ResourceResolver解析出的静态资源，以及通过Http协议暴露的HTTP invoker。
* AnnotationMethodHandlerAdapter是Spring 3.2之前处理@Controller注解类中的@RequestMapping注解方法的适配器。
* RequestMappingHandlerAdapter是Spring3.2之后处理@Controller注解类中的@RequestMapping注解方法的适配器。

> RequestMappingHandlerAdapter与AnnotationMethodHandlerAdapter相比提供了更多的扩展功能，比如Servlet3支持的异步请求，Controller切面支持注解@ControllerAdvice和ControllerAdviceBean等。

## RequestMappingHandlerAdapter

```java
public class RequestMappingHandlerAdapter extends AbstractHandlerMethodAdapter
		implements BeanFactoryAware, InitializingBean {
    ...
    /**
     * RequestMappingHandlerAdapter中handle来自父类AbstractHandlerMethodAdapter
     * 为了减少篇幅我直接把它放在这里。
     */
	public final ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler)
			throws Exception {
        // 调用子类的handleInternal
		return handleInternal(request, response, (HandlerMethod) handler);
	}

    @Override
	protected ModelAndView handleInternal(HttpServletRequest request,
			HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {

		ModelAndView mav;
		checkRequest(request);

		if (this.synchronizeOnSession) {
            // 同步访问Session的情况
			HttpSession session = request.getSession(false);
			if (session != null) {
				Object mutex = WebUtils.getSessionMutex(session);
				synchronized (mutex) {
					mav = invokeHandlerMethod(request, response, handlerMethod);
				}
			}
			else {
				// HttpSession不存在就没不要加同步锁
				mav = invokeHandlerMethod(request, response, handlerMethod);
			}
		}
		else {
			// 直接调用
			mav = invokeHandlerMethod(request, response, handlerMethod);
		}

		if (!response.containsHeader(HEADER_CACHE_CONTROL)) {
			if (getSessionAttributesHandler(handlerMethod).hasSessionAttributes()) {
				applyCacheSeconds(response, this.cacheSecondsForSessionAttributeHandlers);
			}
			else {
				prepareResponse(response);
			}
		}

		return mav;
	}
    protected ModelAndView invokeHandlerMethod(HttpServletRequest request,
			HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {

		ServletWebRequest webRequest = new ServletWebRequest(request, response);
		try {
			WebDataBinderFactory binderFactory = getDataBinderFactory(handlerMethod);
			ModelFactory modelFactory = getModelFactory(handlerMethod, binderFactory);

            // 包装HandlerMethod
			ServletInvocableHandlerMethod invocableMethod = createInvocableHandlerMethod(handlerMethod);
            // argumentResolvers负责处理方法的参数
            // 比如说参数可能是请求参数，也可能是HttpServletRequest,HttpServletResponse,
            // 注解@RequestHeader,@CookieValue,@PathVariable,@RequestBody等。
            // 更多详细内容可以查看HandlerMethodArgumentResolver接口
			invocableMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
            // returnValueHandlers负责处理方法的返回值
            // 返回值可能是viewName,View,Model,ModelAndView,HttpEntity,
            // @ResponseBody返回json,xml,又或者是异步任务等。
            // 更多详细内容可以查看HandlerMethodReturnValueHandler接口
			invocableMethod.setHandlerMethodReturnValueHandlers(this.returnValueHandlers);
            // 创建WebDataBinder,用于将请求参数绑定到JavaBean中,作为方法参数
			invocableMethod.setDataBinderFactory(binderFactory);
            // parameterNameDiscoverer用于获取方法中的请求参数的名字
			invocableMethod.setParameterNameDiscoverer(this.parameterNameDiscoverer);

            // ModelAndViewContainer和ModelAndView类似，都是包装ModelMap和View的。
			ModelAndViewContainer mavContainer = new ModelAndViewContainer();
			mavContainer.addAllAttributes(RequestContextUtils.getInputFlashMap(request));
			modelFactory.initModel(webRequest, mavContainer, invocableMethod);
			mavContainer.setIgnoreDefaultModelOnRedirect(this.ignoreDefaultModelOnRedirect);

			AsyncWebRequest asyncWebRequest = WebAsyncUtils.createAsyncWebRequest(request, response);
			asyncWebRequest.setTimeout(this.asyncRequestTimeout);

            // 异步任务相关的设置
			WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);
			asyncManager.setTaskExecutor(this.taskExecutor);
			asyncManager.setAsyncWebRequest(asyncWebRequest);
			asyncManager.registerCallableInterceptors(this.callableInterceptors);
			asyncManager.registerDeferredResultInterceptors(this.deferredResultInterceptors);

			if (asyncManager.hasConcurrentResult()) {
				Object result = asyncManager.getConcurrentResult();
				mavContainer = (ModelAndViewContainer) asyncManager.getConcurrentResultContext()[0];
				asyncManager.clearConcurrentResult();
				if (logger.isDebugEnabled()) {
					logger.debug("Found concurrent result value [" + result + "]");
				}
				invocableMethod = invocableMethod.wrapConcurrentResult(result);
			}

            // 调用handler
			invocableMethod.invokeAndHandle(webRequest, mavContainer);
			if (asyncManager.isConcurrentHandlingStarted()) {
				return null;
			}

            // 将ModelAndViewContainer转成ModelAndView
			return getModelAndView(mavContainer, modelFactory, webRequest);
		}
		finally {
			webRequest.requestCompleted();
		}
	}
    ...
}
```

# 请求处理方法

***HandlerMethod***

[HandlerMethod](https://docs.spring.io/spring/docs/5.1.4.RELEASE/spring-framework-reference/web.html#mvc-ann-methods)就代表了我们处理请求的方法，也就是我们写在Controller里面有@RequestMapping注解的方法。

![HandlerMethod](http://tva1.sinaimg.cn/large/bda5cd74gy1fskes4ypjgj20eb0etjrl.jpg)

有`bean`和`method`两个属性就能调用这个方法了。

从上面的RequestMappingHandlerAdapter的代码可以看到我们用的是它的子类`ServletInvocableHandlerMethod`，它的继承关系也很简单：

![HandlerMethod](http://tva1.sinaimg.cn/large/bda5cd74gy1fskeub07z5j20d80c0q2y.jpg)

子类InvocableHandlerMethod中有几个重要字段：

* WebDataBinderFactory负责创建DataBinder用于将请求参数绑定方法的参数中
* HandlerMethodArgumentResolverComposite是一组`HandlerMethodArgumentResolver`，用于解析handler方法的参数，比如参数有@PathVariable注解就把请求uri中携带的参数注入进去，如果有@CookieValue注解就把请求Cookie注入，如果有@RequestParam注解就把请求`?`后的参数注入进去...
* ParameterNameDiscoverer负责把handler方法的参数名取出来，方便在没有@RequestParam等任何注解的时候和请求参数的名字匹配。

![HandlerMethod子类的字段](http://tva1.sinaimg.cn/large/bda5cd74gy1fskf4oli68j20nf0bsjrn.jpg)

ServletInvocableHandlerMethod也有一个重要的参数：

* HandlerMethodReturnValueHandlerComposite是一组HandlerMethodReturnValueHandler，他们用来处理handler方法的返回值。比如方法上有@ResponseBody注解就把返回值序列化成JSON或XML响应给客户端。

# 请求处理方法的参数解析和返回值处理

***HandlerMethodArgumentResolver***和***HandlerMethodReturnValueHandler***

HandlerMethodArgumentResolver用于解析请求并注入到方法的参数重，可以通过[官方文档](https://docs.spring.io/spring/docs/5.1.4.RELEASE/spring-framework-reference/web.html#mvc-ann-arguments)查看SpringMVC支持什么类型的参数。

HandlerMethodReturnValueHandler用于处理Controller方法的返回值，可以通过[官方文档](https://docs.spring.io/spring/docs/5.1.4.RELEASE/spring-framework-reference/web.html#mvc-ann-return-types)查看SpringMVC支持什么类型的返回值。

*RequestMappingHandlerAdapter*中有定义了默认的参数解析器和返回值处理器

![默认HandlerMethodArgumentResolver](http://tva1.sinaimg.cn/large/bda5cd74gy1fskghh9aeoj20nr0hl75f.jpg)

![默认HandlerMethodReturnValueHandler](http://tva1.sinaimg.cn/large/bda5cd74gy1fskglkiu9dj20mp0h9dgo.jpg)

# 处理请求分发结果

***DispatcherServlet.processDispatchResult()***

```java
private void processDispatchResult(HttpServletRequest request, HttpServletResponse response,
		HandlerExecutionChain mappedHandler, ModelAndView mv, Exception exception) throws Exception {

	boolean errorView = false;

    // 前面的过程是否出现异常
	if (exception != null) {
		if (exception instanceof ModelAndViewDefiningException) {
			logger.debug("ModelAndViewDefiningException encountered", exception);
            // 用户抛出ModelAndViewDefiningException异常，可能指定特定的错误页面
			mv = ((ModelAndViewDefiningException) exception).getModelAndView();
		}
		else {
            // 对于其他异常交给handlerExceptionResolver异常解析器处理
			Object handler = (mappedHandler != null ? mappedHandler.getHandler() : null);
			mv = processHandlerException(request, response, handler, exception);
			errorView = (mv != null);
		}
	}

	// 请求得到的响应是否返回页面，可能是json,xml等响应
	if (mv != null && !mv.wasCleared()) {
        // 渲染视图
		render(mv, request, response);
		if (errorView) {
			WebUtils.clearErrorRequestAttributes(request);
		}
	}
	else {
		if (logger.isDebugEnabled()) {
			logger.debug("Null ModelAndView returned to DispatcherServlet with name '" + getServletName() +
					"': assuming HandlerAdapter completed request handling");
		}
	}

	if (WebAsyncUtils.getAsyncManager(request).isConcurrentHandlingStarted()) {
		// Concurrent handling started during a forward
		return;
	}

	if (mappedHandler != null) {
		mappedHandler.triggerAfterCompletion(request, response, null);
	}
}
```

# 解析视图名并渲染视图

***DispatcherServlet.render(), resolveViewName()***

```java
protected void render(ModelAndView mv, HttpServletRequest request, HttpServletResponse response) throws Exception {
	// 解析请求的语言
	Locale locale = this.localeResolver.resolveLocale(request);
	response.setLocale(locale);

	View view;
  	// ModelAndView中的View是一个字符串
	if (mv.isReference()) {
        // 视图解析器ViewResolver解析视图的名字，得到View
        // 用的最多的视图解析器就是InternalResourceViewResolver,
        // 它根据返回的viewName加上前缀后缀得到真正的jsp页面的位置。
		view = resolveViewName(mv.getViewName(), mv.getModelInternal(), locale, request);
		if (view == null) {
			throw new ServletException("Could not resolve view with name '" + mv.getViewName() +
					"' in servlet with name '" + getServletName() + "'");
		}
	}
    // ModelAndView中的view就是View类型
	else {
		view = mv.getView();
		if (view == null) {
			throw new ServletException("ModelAndView [" + mv + "] neither contains a view name nor a " +
					"View object in servlet with name '" + getServletName() + "'");
		}
	}

	try {
		if (mv.getStatus() != null) {
			response.setStatus(mv.getStatus().value());
		}
        // 渲染视图
		view.render(mv.getModelInternal(), request, response);
	}
	catch (Exception ex) {
		if (logger.isDebugEnabled()) {
			logger.debug("Error rendering view [" + view + "] in DispatcherServlet with name '" +
					getServletName() + "'", ex);
		}
		throw ex;
	}
}
protected View resolveViewName(String viewName, Map<String, Object> model, Locale locale,
		HttpServletRequest request) throws Exception {

  	// 遍历所有的viewResolver
	for (ViewResolver viewResolver : this.viewResolvers) {
        // 根据viewName解析得到View对象
		View view = viewResolver.resolveViewName(viewName, locale);
		if (view != null) {
			return view;
		}
	}
	return null;
}
```

#视图和视图解析器

***ViewResolver***和***View***

ViewResolver就是把我们在Controller中返回的viewName解析成一个实际的视图层对象；[Spring支持的视图层技术](https://docs.spring.io/spring/docs/5.0.3.BUILD-SNAPSHOT/spring-framework-reference/web.html#mvc-view)包括JSP，FreeMarker，Velocity，Groovy，Tiles，PDF，Excel等，还有就是Spring官方比较推荐的Thymeleaf(Spring没有对Thymeleaf直接支持，需要导入Thymeleaf的jar包)。

![ViewResolver](http://img-blog.csdn.net/20171226163906852?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

这些视图解析器解析viewName得到如下的视图对象：

![View](http://img-blog.csdn.net/20171226163925347?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

## InternalResourceViewResolver

InternalResourceViewResolver是用的最多ViewResolver。

想要分析InternalResourceViewResolver，我们还需要看看它父类的代码(resolveViewName方法在父类中)：

![InternalResourceViewResolver](http://img-blog.csdn.net/20171226163942477?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

**AbstractCachingViewResolver**

```java
/**
 * 所有的视图层技术都会做缓存
 */
public abstract class AbstractCachingViewResolver extends WebApplicationObjectSupport implements ViewResolver {
    /** 默认最大缓存数为1024 */
	public static final int DEFAULT_CACHE_LIMIT = 1024;
  	/** 用于标记缓存中解析失败的viewName */
	private static final View UNRESOLVED_VIEW = new View() {
		public String getContentType() {
			return null;
		}
		public void render(Map<String, ?> model, HttpServletRequest request, HttpServletResponse response) {
		}
	};
    /** 最大缓存数 */
	private volatile int cacheLimit = DEFAULT_CACHE_LIMIT;

	/** 如果第一次未解析成功，下一次是否再去解析 */
	private boolean cacheUnresolved = true;

	/** 视图缓存，加快视图解析速度
     * 将ConcurrentHashMap的无锁特性以及LinkedHashMap的缓存特性结合起来使用
     */
    /** 无锁缓存 */
	private final Map<Object, View> viewAccessCache = new ConcurrentHashMap<Object, View>(DEFAULT_CACHE_LIMIT);
    /** LRU缓存 */
	private final Map<Object, View> viewCreationCache =
			new LinkedHashMap<Object, View>(DEFAULT_CACHE_LIMIT, 0.75f, true) {
				protected boolean removeEldestEntry(Map.Entry<Object, View> eldest) {
					if (size() > getCacheLimit()) {
                        // 当缓存容量达到上限，清除最早未使用的View
						viewAccessCache.remove(eldest.getKey());// 把无锁缓存中的数据清除
						return true; // 返回true，清除Lru缓存数据
					} else {
						return false;
					}
				}
			};
    public View resolveViewName(String viewName, Locale locale) throws Exception {
		if (!isCache()) {
            // 不允许缓存就直接创建视图
			return createView(viewName, locale);
		}
		else {
            /**
             *  直接把视图名和地区连成字符串作为key
             *	protected Object getCacheKey(String viewName, Locale locale) {
			 *		return viewName + '_' + locale;
			 *	}
             */
			Object cacheKey = getCacheKey(viewName, locale);
			View view = this.viewAccessCache.get(cacheKey);
			if (view == null) {
                // 无锁缓存中没有命中，从Lru缓存中查找
				synchronized (this.viewCreationCache) { // 多线程加同步锁
					view = this.viewCreationCache.get(cacheKey);
					if (view == null) {
						// 让子类创建视图对象
						view = createView(viewName, locale);
						if (view == null && this.cacheUnresolved) {
                            // 视图解析失败，下次就不解析了，放一个UNRESOLVED_VIEW标志到缓存中
							view = UNRESOLVED_VIEW;
						}
						if (view != null) {
                            // 放入无锁缓存
							this.viewAccessCache.put(cacheKey, view);
                            // 放入Lru缓存
							this.viewCreationCache.put(cacheKey, view);
							if (logger.isTraceEnabled()) {
								logger.trace("Cached view [" + cacheKey + "]");
							}
						}
					}
				}
			}
            // 如果是标记的UNRESOLVED_VIEW，就返回null，让下一个视图解析器解析
			return (view != UNRESOLVED_VIEW ? view : null);
		}
	}
  	protected View createView(String viewName, Locale locale) throws Exception {
		return loadView(viewName, locale);
	}
    // 留给子类实现
    protected abstract View loadView(String viewName, Locale locale) throws Exception;
}
```

**UrlBasedViewResolver**

```java
public class UrlBasedViewResolver extends AbstractCachingViewResolver implements Ordered {

    /** 返回字符串时可以加的两个前缀，SpringMVC会更具这两个前缀选用不同的View */
	public static final String REDIRECT_URL_PREFIX = "redirect:";
	public static final String FORWARD_URL_PREFIX = "forward:";

    /** 这就是我们在InternalResourceViewResolver中配置的前后缀 */
  	private String prefix = "";
	private String suffix = "";

    // 配置通配符，代表这个ViewResolver可接受的viewName
    private String[] viewNames;
    // order代表了ViewResolver的优先级，值越小优先级越高
    // 这里的order默认是Integer.MAX_VALUE，也就是优先级最低
	private int order = Integer.MAX_VALUE;

    protected View createView(String viewName, Locale locale) throws Exception {
		if (!canHandle(viewName, locale)) {
            // 检查viewName是否被viewNames接受
			return null;
		}
		// 检查"redirect:"前缀
		if (viewName.startsWith(REDIRECT_URL_PREFIX)) {
			String redirectUrl = viewName.substring(REDIRECT_URL_PREFIX.length());
            // "redirect:"创建RedirectView
			RedirectView view = new RedirectView(redirectUrl, isRedirectContextRelative(), isRedirectHttp10Compatible());
			view.setHosts(getRedirectHosts());
			return applyLifecycleMethods(viewName, view);
		}
		// 检查"forward:"前缀
		if (viewName.startsWith(FORWARD_URL_PREFIX)) {
			String forwardUrl = viewName.substring(FORWARD_URL_PREFIX.length());
            // "forward:"前缀创建InternalResourceView
			return new InternalResourceView(forwardUrl);
		}
		// 否则调用父类createView,让其回调loadView
		return super.createView(viewName, locale);
	}
    protected View loadView(String viewName, Locale locale) throws Exception {
        // 根据viewName构造View
		AbstractUrlBasedView view = buildView(viewName);
		View result = applyLifecycleMethods(viewName, view);
		return (view.checkResource(locale) ? result : null);
	}
    protected AbstractUrlBasedView buildView(String viewName) throws Exception {
        // 子类中设置ViewClass
		AbstractUrlBasedView view = (AbstractUrlBasedView) BeanUtils.instantiateClass(getViewClass());
        // 加前缀,加后缀,得到真实路径
		view.setUrl(getPrefix() + viewName + getSuffix());

		String contentType = getContentType();
		if (contentType != null) {
			view.setContentType(contentType);
		}

		view.setRequestContextAttribute(getRequestContextAttribute());
		view.setAttributesMap(getAttributesMap());

		Boolean exposePathVariables = getExposePathVariables();
		if (exposePathVariables != null) {
			view.setExposePathVariables(exposePathVariables);
		}
		Boolean exposeContextBeansAsAttributes = getExposeContextBeansAsAttributes();
		if (exposeContextBeansAsAttributes != null) {
			view.setExposeContextBeansAsAttributes(exposeContextBeansAsAttributes);
		}
		String[] exposedContextBeanNames = getExposedContextBeanNames();
		if (exposedContextBeanNames != null) {
			view.setExposedContextBeanNames(exposedContextBeanNames);
		}

		return view;
	}
}
```

**InternalResourceViewResolver**

```java
public class InternalResourceViewResolver extends UrlBasedViewResolver {
    // 检查类路径下是否添加了JSTL的依赖包
	private static final boolean jstlPresent = ClassUtils.isPresent(
			"javax.servlet.jsp.jstl.core.Config", InternalResourceViewResolver.class.getClassLoader());

    // 使用include还是使用forward,
    // 默认false,也就是默认使用forward
	private Boolean alwaysInclude;

	public InternalResourceViewResolver() {
        // 如果类路径下有JSTL的依赖包则使用JstlView，否则使用InternalResourceView
		Class<?> viewClass = requiredViewClass();
		if (InternalResourceView.class == viewClass && jstlPresent) {
			viewClass = JstlView.class;
		}
		setViewClass(viewClass);
	}

	public InternalResourceViewResolver(String prefix, String suffix) {
		this();
		setPrefix(prefix);
		setSuffix(suffix);
	}

	@Override
	protected Class<?> requiredViewClass() {
		return InternalResourceView.class;
	}

	public void setAlwaysInclude(boolean alwaysInclude) {
		this.alwaysInclude = alwaysInclude;
	}


	@Override
	protected AbstractUrlBasedView buildView(String viewName) throws Exception {
		InternalResourceView view = (InternalResourceView) super.buildView(viewName);
		if (this.alwaysInclude != null) {
			view.setAlwaysInclude(this.alwaysInclude);
		}
		view.setPreventDispatchLoop(true);
		return view;
	}

}
```

## InternalResourceView，RedirectView，JstlView

前面ViewResolver中涉及到了三个View的实现类：InternalResourceView，RedirectView，JstlView

![InternalResourceView](http://img-blog.csdn.net/20171226164020012?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

**AbstractView**

```java
public abstract class AbstractView extends WebApplicationObjectSupport implements View, BeanNameAware {
    ...
	public void render(Map<String, ?> model, HttpServletRequest request, HttpServletResponse response) throws Exception {
		if (logger.isTraceEnabled()) {
			logger.trace("Rendering view with name '" + this.beanName + "' with model " + model +
				" and static attributes " + this.staticAttributes);
		}
        // 合并request,model中的数据
		Map<String, Object> mergedModel = createMergedOutputModel(model, request, response);
        // 设置响应头
		prepareResponse(request, response);
        // 将model中的数据渲染到视图上
		renderMergedOutputModel(mergedModel, getRequestToExpose(request), response);
	}
    // 由子类实现该方法
  	protected abstract void renderMergedOutputModel(
			Map<String, Object> model, HttpServletRequest request, HttpServletResponse response) throws Exception;
}
```

**InternalResourceView**

```java
public class InternalResourceView extends AbstractUrlBasedView {
    ...
    protected void renderMergedOutputModel(
			Map<String, Object> model, HttpServletRequest request, HttpServletResponse response) throws Exception {

		// 将model中的数据作为request的Attribute
		exposeModelAsRequestAttributes(model, request);

		// 我们可以重载InternalResourceView.exposeHelpers方法
        // 添加一些工具方法
        // JstlView就是重载了这个方法
		exposeHelpers(request);

		// 得到分发的路径
		String dispatcherPath = prepareForRendering(request, response);

		// 获取javax.servlet.RequestDispatcher
		RequestDispatcher rd = getRequestDispatcher(request, dispatcherPath);
		if (rd == null) {
			throw new ServletException("Could not get RequestDispatcher for [" + getUrl() +
					"]: Check that the corresponding file exists within your web application archive!");
		}

		// If already included or response already committed, perform include, else forward.
		if (useInclude(request, response)) {
            // RequestDispatcher.include
			rd.include(request, response);
		}

		else {
            // RequestDispatcher.forward
			rd.forward(request, response); // forward
		}
	}
}
```

**RedirectView**

```java
public class RedirectView extends AbstractUrlBasedView implements SmartView {
    ...
	protected void renderMergedOutputModel(Map<String, Object> model, HttpServletRequest request,
			HttpServletResponse response) throws IOException {
		...
		sendRedirect(request, response, targetUrl, this.http10Compatible);
	}
	protected void sendRedirect(HttpServletRequest request, HttpServletResponse response,
			String targetUrl, boolean http10Compatible) throws IOException {

      	// HttpServletResponse.encodeRedirectURL()用于将sessionId编码到url中
		String encodedURL = (isRemoteHost(targetUrl) ? targetUrl : response.encodeRedirectURL(targetUrl));
		if (http10Compatible) {
			HttpStatus attributeStatusCode = (HttpStatus) request.getAttribute(View.RESPONSE_STATUS_ATTRIBUTE);
			if (this.statusCode != null) {
				response.setStatus(this.statusCode.value());
				response.setHeader("Location", encodedURL);
			}
			else if (attributeStatusCode != null) {
				response.setStatus(attributeStatusCode.value());
				response.setHeader("Location", encodedURL);
			}
			else {
				// 默认的302重定向状态码
				response.sendRedirect(encodedURL);
			}
		}
		else {
			HttpStatus statusCode = getHttp11StatusCode(request, response, targetUrl);// Http1.1
			response.setStatus(statusCode.value());
			response.setHeader("Location", encodedURL);
		}
	}
}
```

