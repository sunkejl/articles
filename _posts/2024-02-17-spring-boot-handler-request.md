---
layout: post
title: "当一个请求进来的时候, Spring Boot 是如何处理的？"
date: 2024-02-17 10:00:00 +0800
tags: [Jekyll, Markdown, 博客]
author: "孙珂"
---


# 当一个请求进来的时候, Spring Boot 是如何处理的？

`1.` 在 Spring Boot 启动阶段，把含有 @RequestMapping 的方法加入到 MappingRegistry#registry 的Map对像中。

`2.` 当 Spring Boot 启动完成，外部请求进入的入口方法是 DispatcherServlet 的 doService() 方法。

也就是，打印MappingRegistry#registry 这个Map对像，可以得到所有的 path 对应的 Controller。

##  Spring Boot 启动阶段

RequestMappingHandlerMapping 在初始化时，

会调用 afterPropertiesSet()，处理 RequestMapping 中的相关Methods。

initHandlerMethods() 负责遍历 ApplicationContext 中的 Bean。

```java
	/**
     * org.springframework.web.servlet.handler.AbstractHandlerMethodMapping#afterPropertiesSet
     * 
	 * Detects handler methods at initialization.
	 */
	@Override
	public void afterPropertiesSet() {
		initHandlerMethods();
	}

    /**
     * org.springframework.web.servlet.handler.AbstractHandlerMethodMapping#initHandlerMethods
     *
     * Scan beans in the ApplicationContext, detect and register handler methods.
     */
    protected void initHandlerMethods() {
        for (String beanName : getCandidateBeanNames()) {
            if (!beanName.startsWith(SCOPED_TARGET_NAME_PREFIX)) {
                processCandidateBean(beanName);
            }
        }
        handlerMethodsInitialized(getHandlerMethods());
    }
```


processCandidateBean() 过滤出含有 @Controller 和 @RequestMapping 注解的 Bean 进行处理

detectHandlerMethods() 调用 registerHandlerMethod() 组装 MappingRegistry#registry 的Map对像

```java


	/**
     * org.springframework.web.servlet.handler.AbstractHandlerMethodMapping#processCandidateBean
     * 
     * Determine the type of the specified candidate bean and call
     * {@link #detectHandlerMethods} if identified as a handler type.
     */
	protected void processCandidateBean(String beanName) {
		if (beanType != null && isHandler(beanType)) {
			detectHandlerMethods(beanName);
		}
	}

	/**
     * org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping#isHandler
     *
	 * Expects a handler to have either a type-level @{@link Controller}
	 * annotation or a type-level @{@link RequestMapping} annotation.
	 */
	@Override
	protected boolean isHandler(Class<?> beanType) {
		return (AnnotatedElementUtils.hasAnnotation(beanType, Controller.class) ||
				AnnotatedElementUtils.hasAnnotation(beanType, RequestMapping.class));
	}

    /**
     * org.springframework.web.servlet.handler.AbstractHandlerMethodMapping#registerHandlerMethod
     * 
     * Register a handler method and its unique mapping. Invoked at startup for
     * each detected handler method.
     */
    protected void registerHandlerMethod(Object handler, Method method, T mapping) {
        this.mappingRegistry.register(mapping, handler, method);
    }
```


## 当 Spring Boot 启动完成

request请求进来的时候，会调用 DispatcherServlet 的 doService() 方法。

doDispatch() 会找到对应的 mappedHandler，然后获得 HandlerAdapter，调用对应的 handle() 方法。 


```java
	/**
     * org.springframework.web.servlet.DispatcherServlet#doService
     *
	 * Exposes the DispatcherServlet-specific request attributes and delegates to {@link #doDispatch}
	 * for the actual dispatching.
	 */
	@Override
	protected void doService(HttpServletRequest request, HttpServletResponse response) throws Exception {
        ...
        doDispatch(request, response);
        ...
    }

    /**
     * Process the actual dispatching to the handler.
     * <p>The handler will be obtained by applying the servlet's HandlerMappings in order.
     * The HandlerAdapter will be obtained by querying the servlet's installed HandlerAdapters
     * to find the first that supports the handler class.
     * <p>All HTTP methods are handled by this method. It's up to HandlerAdapters or handlers
     * themselves to decide which methods are acceptable.
     */
    protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
        ...
        mappedHandler = getHandler(processedRequest);
        ...
        // Determine handler adapter for the current request.
        HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());
        ...
        // Actually invoke the handler.
        mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
        ...
    }
    
    /**
     * Return the HandlerExecutionChain for this request.
     * <p>Tries all handler mappings in order.
     */
    protected HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
        if (this.handlerMappings != null) {
            for (HandlerMapping mapping : this.handlerMappings) {
                HandlerExecutionChain handler = mapping.getHandler(request);
                if (handler != null) {
                    return handler;
                }
            }
        }
        return null;
    }
    
    /**
     * org.springframework.web.servlet.handler.AbstractHandlerMapping#getHandler
     *
     * Look up a handler for the given request, falling back to the default
     * handler if no specific one is found.
     */
    
    public final HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
        Object handler = getHandlerInternal(request);
    }
    
    
    /**
     * Look up a handler method for the given request.
     */
    protected HandlerMethod getHandlerInternal(HttpServletRequest request) throws Exception {
        String lookupPath = initLookupPath(request);
        this.mappingRegistry.acquireReadLock();
        try {
            HandlerMethod handlerMethod = lookupHandlerMethod(lookupPath, request);
            return (handlerMethod != null ? handlerMethod.createWithResolvedBean() : null);
        }
        finally {
            this.mappingRegistry.releaseReadLock();
        }
    }
    
    /**
     * org.springframework.web.servlet.handler.AbstractHandlerMethodMapping#lookupHandlerMethod
     * 
     * Look up the best-matching handler method for the current request.
     * If multiple matches are found, the best match is selected.
     */
    @Nullable
    protected HandlerMethod lookupHandlerMethod(String lookupPath, HttpServletRequest request) throws Exception {
        
        return bestMatch.getHandlerMethod();
    }
    
    
    
    /**
     * org.springframework.web.servlet.handler.AbstractHandlerMethodMapping#addMatchingMappings
     
     * 从this.mappingRegistry.getRegistrations() 拿到对应的controller
     */
    private void addMatchingMappings(Collection<T> mappings, List<Match> matches, HttpServletRequest request) {
        for (T mapping : mappings) {
            T match = getMatchingMapping(mapping, request);
            if (match != null) {
                matches.add(new Match(match, this.mappingRegistry.getRegistrations().get(mapping)));
            }
        }
    }
    
    /**
     * Checks if all conditions in this request mapping info match the provided
     * request and returns a potentially new request mapping info with conditions
     * tailored to the current request.
     * <p>For example the returned instance may contain the subset of URL
     * patterns that match to the current request, sorted with best matching
     * patterns on top.
     * 构建 RequestMappingInfo 对象
     */
    public RequestMappingInfo getMatchingCondition(HttpServletRequest request) {
        ...
        return new RequestMappingInfo(this.name, pathPatterns, patterns,
                methods, params, headers, consumes, produces, custom, this.options);
    
    }
    
```
    

当查找到对应的 handler 后， 构建参数，最终通过反射，调用对应的 Controller 对应的方法并返回结果
    
    
```java
    /**
     * org.springframework.web.servlet.mvc.method.AbstractHandlerMethodAdapter#handle
     * This implementation expects the handler to be an {@link HandlerMethod}.
     */
    public final ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler)
            throws Exception {
    
        return handleInternal(request, response, (HandlerMethod) handler);
    }
    
    
    protected ModelAndView handleInternal(HttpServletRequest request,
                                          HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {
        // No synchronization on session demanded at all...
        mav = invokeHandlerMethod(request, response, handlerMethod);
    
    }
    
    /**
     * org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter#invokeHandlerMethod
     *
     * Invoke the {@link RequestMapping} handler method preparing a {@link ModelAndView}
     * if view resolution is required.
     */
    protected ModelAndView invokeHandlerMethod(HttpServletRequest request,
                                               HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {
        
        invocableMethod.invokeAndHandle(webRequest, mavContainer);
    
    }
    
    
    /**
     * Invoke the method and handle the return value through one of the
     * configured {@link HandlerMethodReturnValueHandler HandlerMethodReturnValueHandlers}.
     */
    public void invokeAndHandle(ServletWebRequest webRequest, ModelAndViewContainer mavContainer,
                                Object... providedArgs) throws Exception {
    
        Object returnValue = invokeForRequest(webRequest, mavContainer, providedArgs);
        ...
    }
    
    
    /**
     * Invoke the method after resolving its argument values in the context of the given request.
     * <p>Argument values are commonly resolved through
     * {@link HandlerMethodArgumentResolver HandlerMethodArgumentResolvers}.
     * The {@code providedArgs} parameter however may supply argument values to be used directly,
     * i.e. without argument resolution. Examples of provided argument values include a
     * {@link WebDataBinder}, a {@link SessionStatus}, or a thrown exception instance.
     * Provided argument values are checked before argument resolvers.
     * <p>Delegates to {@link #getMethodArgumentValues} and calls {@link #doInvoke} with the
     * resolved arguments.
     */
    public Object invokeForRequest(NativeWebRequest request, @Nullable ModelAndViewContainer mavContainer,
                                   Object... providedArgs) throws Exception {
    
        Object[] args = getMethodArgumentValues(request, mavContainer, providedArgs);
        if (logger.isTraceEnabled()) {
            logger.trace("Arguments: " + Arrays.toString(args));
        }
        return doInvoke(args);
    }
    
    
    /**
     * org.springframework.web.method.support.InvocableHandlerMethod#doInvoke
     * use java.lang.reflect.Method.invoke
     *
     * Invoke the handler method with the given argument values.
     */
    protected Object doInvoke(Object... args) throws Exception {
        ...
        return method.invoke(getBean(), args);
    }


```







