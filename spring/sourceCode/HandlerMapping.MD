#HandlerMapping源码解析
##1.HandlerMapping功能分析
HandlerMapping是spring中最重要的一个类,主要功能就是为请求找到合适的处理器，现在Controller只是处理器的一种，目前我们自己公司自己实现了一套框架，前端js可以直接调用后端 soa service，忽略controller的存在，适合一些无用户状态的场景。

先来介绍下目前spring 已经提供的handlermapping

1.   BeanNameUrlHandlerMapping

>请求路径到bean name的映射

2.   SimpleUrlHandlerMapping

>最原始的spring使用方式，根struts差不多，通过配置请求路径对应处理的controller类，如下面例子
```xml
<bean id="simpleUrlMapping" class="org.springframework.web.servlet.handler.SimpleUrlHandlerMapping">
<property name="mappings">
<props>
    <prop key="/listCourses.go">listCoursesController</prop>
    <prop key="/login.go">loginController</prop>
<props>
<property>
<bean>
```

3.   ControllerBeanNameHandlerMapping

>This is similar to BeanNameUrlHandlerMapping but doesn't expect bean names to follow the URL convention: It turns plain bean names into URLs by prepending a slash and optionally applying a specified prefix and/or suffix. However, it only does so for well-known controller types, as listed above (analogous to ControllerClassNameHandlerMapping).

4.   ControllerClassNameHandlerMapping

>这个主要是类名映射，看看下面就知道了
* WelcomeController -> /welcome*
* HomeController -> /home*
>这种方式通常搭配着`MultiActionController` ，比如/abc/to_add.do 这个表示调用AbcController.java下的to_add的方法。前提是AbcController.java继承MultiActionController  

5.  RequestMappingHandlerMapping（3.2之前是使用DefaultAnnotationHandlerMapping 这个去处理，但是后来弃用了）

>这个就是现在默认的请求匹配方式，通过@RequestMapping 注解来决定调用方法和具体的类，也是最常用的一种。

DispatcherServlet初始化的时候会调用`initHandlerMappings()`进行初始化

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
  //初始化handlermapping
  private void initHandlerMappings(ApplicationContext context) {
		this.handlerMappings = null;
		if (this.detectAllHandlerMappings) {
			// Find all HandlerMappings in the ApplicationContext, including ancestor contexts.
			Map<String, HandlerMapping> matchingBeans =
					BeanFactoryUtils.beansOfTypeIncludingAncestors(context, HandlerMapping.class, true, false);
			if (!matchingBeans.isEmpty()) {
				this.handlerMappings = new ArrayList<HandlerMapping>(matchingBeans.values());
				// We keep HandlerMappings in sorted order.
				//通过@order注解去排序
				AnnotationAwareOrderComparator.sort(this.handlerMappings);
			}
		}
		else {
			try {
				HandlerMapping hm = context.getBean(HANDLER_MAPPING_BEAN_NAME, HandlerMapping.class);
				this.handlerMappings = Collections.singletonList(hm);
			}
			catch (NoSuchBeanDefinitionException ex) {
				// Ignore, we'll add a default HandlerMapping later.
			}
		}

		// Ensure we have at least one HandlerMapping, by registering
		// a default HandlerMapping if no other mappings are found.
		if (this.handlerMappings == null) {
			this.handlerMappings = getDefaultStrategies(context, HandlerMapping.class);
			if (logger.isDebugEnabled()) {
				logger.debug("No HandlerMappings found in servlet '" + getServletName() + "': using default");
			}
		}
	}
```

>下面是初始化之后的一个状态，我这边使用的是spring boot启动，大致会初始化10个handlermapping，如下图所示：

![Renderings](http://git.oschina.net/wangkang_daydayup/Learning_Summary/raw/master/spring/sourceCode/img/HandlerMapping/QQ20160517-0.png?dir=0&filepath=spring%2FsourceCode%2Fimg%2FHandlerMapping%2FQQ20160517-0.png&oid=d6cda8a923e06f8ff75ad6c09a7602064412a054&sha=1c00a4cdcc2e22daf2a832b64d1929b46ffa71b7)

>下图可以看到，默认的`RequestMappingHandlerMapping`已经把所有路径映射进去,相关的spring拦截器也放到了`RequestMappingHandlerMapping`中，由此说来，拦截器其实是跟`RequestMappingHandlerMapping`进行关联的。

![Renderings](http://git.oschina.net/wangkang_daydayup/Learning_Summary/raw/master/spring/sourceCode/img/HandlerMapping/QQ20160517-1.png?dir=0&filepath=spring%2FsourceCode%2Fimg%2FHandlerMapping%2FQQ20160517-1.png&oid=d9a014a41938f86431105c6e132fedfd319b72f0&sha=1c00a4cdcc2e22daf2a832b64d1929b46ffa71b7)

##2. HandlerMapping流程分析

HandlerMapping流程需要弄清楚几个类的定义:

1.   HandlerExecutionChain

(Handler execution chain, consisting of handler object and any handler interceptors. Returned by HandlerMapping's)根据英文的翻译来说`HandlerExecutionChain`是 handler的执行链，由一些handler object 和 handler interceptors组成,这个里面包含了interceptor的所有执行，包括这个请求是否通过preHandle，大致逻辑如下

```java

		 boolean applyPreHandle(HttpServletRequest request, HttpServletResponse response) throws Exception {
			 //获取所有拦截器进行遍历
			HandlerInterceptor[] interceptors = getInterceptors();
			if (!ObjectUtils.isEmpty(interceptors)) {
				for (int i = 0; i < interceptors.length; i++) {
					HandlerInterceptor interceptor = interceptors[i];
					//判断拦截器是否可以继续执行
					if (!interceptor.preHandle(request, response, this.handler)) {
						triggerAfterCompletion(request, response, null);
						return false;
					}
					this.interceptorIndex = i;
				}
			}
			return true;
		}

```

2.   HandlerAdapter

`HandlerAdapter` 这个是spring的核心功能，所有的请求对应Controller方法都是通过HandlerAdapter 进行映射和调用并且返回结果
