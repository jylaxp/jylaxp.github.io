---
layout: post
title: Spring 初始化 ContextLoaderListener 与 DispatcherServlet
category: Spring
img-path: 2017-11-22-ContextLoaderListener-And-DispatcherServlet
---
# Spring 初始化 ContextLoaderListener 与 DispatcherServlet
## 0. 概述
分析web项目中spring的配置，初始化，bean解析和注入的过程，以及spring bean 的生命周期和扩展点，深入学习spring框架的使用。  
示例项目: [springinside](https://github.com/jylaxp/springinside)

## 1. spring web 配置文件
项目常见的spring web 的web.xml， 这里去掉了一些无关的配置，尽量最小化配置
```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app version="2.4"
         xmlns="http://java.sun.com/xml/ns/j2ee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://java.sun.com/xml/ns/j2ee http://java.sun.com/xml/ns/j2ee/web-app_2_4.xsd">

    <context-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>classpath*:META-INF/spring/*.xml</param-value>
    </context-param>

    <listener>
        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
    </listener>

    <servlet>
        <description>spring inside</description>
        <display-name>spring inside</display-name>
        <servlet-name>springinside</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <init-param>
            <param-name>contextConfigLocation</param-name>
            <param-value>WEB-INF/applicationContext.xml</param-value>
        </init-param>
        <load-on-startup>1</load-on-startup>
    </servlet>

    <servlet-mapping>
        <servlet-name>springinside</servlet-name>
        <url-pattern>/*</url-pattern>
    </servlet-mapping>
</web-app>
```

1. 配置context-param,contextConfigLocation 指定ContextLoaderListener使用的配置文件
2. 配置了ContextLoaderListener，监听web容器的启动，用于做些初始化工作
3. 配置了DispatcherServlet处理web请求，指定了DispatcherServlet要加载的配置文件  

对于spring web 项目来说，其中1和2不是必须的，可以没有。但是，这里配置ContextLoaderListener，是为了和部门的项目一致。部门的sof开发框架在web容器启动的时候有一些初始化操作，就是在ContextLoaderListener这里触发的。接下来我们分析ContextLoaderListener和DispatcherServlet源码，来看看spring是怎么完成初始化的。

## 2. ContextLoaderListener分析
### 2.1 继承关系
首先，看一下ContextLoaderListener的继承关系

![]({{site.url}}/{{site.blog-img}}/{{page.img-path}}/1.png)

可以看到，ContextLoaderListener实现了ServletContextListener接口。ServletContextListener接口够监听 ServletContext 对象的生命周期，实际上就是监听 Web应用的生命周期。当Servlet 容器启动或终止Web 应用时，会触发ServletContextEvent 事件，该事件由ServletContextListener 来处理。
```java
public interface ServletContextListener extends EventListener {
	/**
	 ** Notification that the web application initialization
	 ** process is starting.
	 ** All ServletContextListeners are notified of context
	 ** initialization before any filter or servlet in the web
	 ** application is initialized.
	 */

    public void contextInitialized ( ServletContextEvent sce );

	/**
	 ** Notification that the servlet context is about to be shut down.
	 ** All servlets and filters have been destroy()ed before any
	 ** ServletContextListeners are notified of context
	 ** destruction.
	 */
    public void contextDestroyed ( ServletContextEvent sce );
}
```
当Servlet启动时就会调用contextInitialized方法；当Servlet销毁时会调用contextDestroyed方法。  
### 2.2 ContextLoaderListener
接下来我们看ContextLoaderListener源码。
```java
public class ContextLoaderListener extends ContextLoader implements ServletContextListener {

	public ContextLoaderListener() {
	}


	public ContextLoaderListener(WebApplicationContext context) {
		super(context);
	}


	/**
	 * Initialize the root web application context.
	 */
	@Override
	public void contextInitialized(ServletContextEvent event) {
		initWebApplicationContext(event.getServletContext());
	}


	/**
	 * Close the root web application context.
	 */
	@Override
	public void contextDestroyed(ServletContextEvent event) {
		closeWebApplicationContext(event.getServletContext());
		ContextCleanupListener.cleanupAttributes(event.getServletContext());
	}

}
```
ContextLoaderListener的代码很简单，在Servlet启动时，调用initWebApplicationContext方法初始化WebApplicationContext这个web应用上下文；在Servlet销毁时，关闭应用上下文。Spring是怎么初始化web应用上下文的，我们跟踪代码进入initWebApplicationContext方法一探究竟。
### 2.3 initWebApplicationContext 分析
```java
public WebApplicationContext initWebApplicationContext(ServletContext servletContext) {
		if (servletContext.getAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE) != null) {
			throw new IllegalStateException(
					"Cannot initialize context because there is already a root application context present - " +
					"check whether you have multiple ContextLoader* definitions in your web.xml!");
		}

		Log logger = LogFactory.getLog(ContextLoader.class);
		servletContext.log("Initializing Spring root WebApplicationContext");
		if (logger.isInfoEnabled()) {
			logger.info("Root WebApplicationContext: initialization started");
		}
		long startTime = System.currentTimeMillis();

		try {
			// Store context in local instance variable, to guarantee that
			// it is available on ServletContext shutdown.
			if (this.context == null) {
				this.context = createWebApplicationContext(servletContext);
			}
			if (this.context instanceof ConfigurableWebApplicationContext) {
				ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) this.context;
				if (!cwac.isActive()) {
					// The context has not yet been refreshed -> provide services such as
					// setting the parent context, setting the application context id, etc
					if (cwac.getParent() == null) {
						// The context instance was injected without an explicit parent ->
						// determine parent for root web application context, if any.
						ApplicationContext parent = loadParentContext(servletContext);
						cwac.setParent(parent);
					}
					configureAndRefreshWebApplicationContext(cwac, servletContext);
				}
			}
			servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, this.context);

			ClassLoader ccl = Thread.currentThread().getContextClassLoader();
			if (ccl == ContextLoader.class.getClassLoader()) {
				currentContext = this.context;
			}
			else if (ccl != null) {
				currentContextPerThread.put(ccl, this.context);
			}

			if (logger.isDebugEnabled()) {
				logger.debug("Published root WebApplicationContext as ServletContext attribute with name [" +
						WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE + "]");
			}
			if (logger.isInfoEnabled()) {
				long elapsedTime = System.currentTimeMillis() - startTime;
				logger.info("Root WebApplicationContext: initialization completed in " + elapsedTime + " ms");
			}

			return this.context;
		}
		catch (RuntimeException ex) {
			logger.error("Context initialization failed", ex);
			servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, ex);
			throw ex;
		}
		catch (Error err) {
			logger.error("Context initialization failed", err);
			servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, err);
			throw err;
		}
	}
```
initWebApplicationContext方法一共做了三件事：
##### 1. 如果this.context为null, 则创建WebApplicationContext
```java
if (this.context == null) {
	this.context = createWebApplicationContext(servletContext);
}
```
我们在web.xml里，只配置了一个Listener，所以在ContextLoaderListener之前没有创建过WebApplicationContext，固这里this.context为null，所以会创建WebApplicationContext。
##### 2. 如果this.context是ConfigurableWebApplicationContext的实例，并且没有刷新过WebApplicationContext，则配置并刷新WebApplicationContext。
```java
if (this.context instanceof ConfigurableWebApplicationContext) {
	ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) this.context;
	if (!cwac.isActive()) {
		// The context has not yet been refreshed -> provide services such as
		// setting the parent context, setting the application context id, etc
		if (cwac.getParent() == null) {
			// The context instance was injected without an explicit parent ->
			// determine parent for root web application context, if any.
			ApplicationContext parent = loadParentContext(servletContext);
			cwac.setParent(parent);
		}
		configureAndRefreshWebApplicationContext(cwac, servletContext);
	}
}
```
什么意思呢？首先，我们看看this.context的起继承关系 。

![]({{site.url}}/{{site.blog-img}}/{{page.img-path}}/2.png)

this.contexnt的类型是WebApplicationContext， 可以看到WebApplicationContext是个接口，它继承了BeanFactory。this.context是Spring的IoC容器，所以下面的配置和刷新应用上下文就不足为奇了，因为它要解析，注入bean。所以configureAndRefreshWebApplicationContext方法，会完成bean解析，加载，注入的过程。现在我们知道了WebApplicationContext是一个接口，这里的this.context到底会是什么那个具体的实例呢？其实，这里的this.context是XmlWebApplicationContext的具体实例。我们看一下XmlWebApplicationContext的继承关系。


![]({{site.url}}/{{site.blog-img}}/{{page.img-path}}/3.png)


可以看到，XmlWebApplicationContext实现了ConfigurableWebApplicationContext 接口，ConfigurableWebApplicationContext 继承了WebApplicationContext。继承关系理清楚了，这里的代码也就很容易看懂了。


##### 3. 将WebApplicationContext放入servletContext中，供后续流程使用
```java
servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, this.context);
```
这里讲this.context放入servletContext中，后续的DispatcherServlet会从servletContext取出该容器，作为其创建的容器的父容器。

### 2.4 configureAndRefreshWebApplicationContext 分析
现在我们进入configureAndRefreshWebApplicationContext方法，看看是如何配置和刷新上下文的。
```java
protected void configureAndRefreshWebApplicationContext(ConfigurableWebApplicationContext wac, ServletContext sc) {
	if (ObjectUtils.identityToString(wac).equals(wac.getId())) {
		// The application context id is still set to its original default value
		// -> assign a more useful id based on available information
		String idParam = sc.getInitParameter(CONTEXT_ID_PARAM);
		if (idParam != null) {
			wac.setId(idParam);
		}
		else {
			// Generate default id...
			wac.setId(ConfigurableWebApplicationContext.APPLICATION_CONTEXT_ID_PREFIX +
					ObjectUtils.getDisplayString(sc.getContextPath()));
		}
	}

	wac.setServletContext(sc);
	String configLocationParam = sc.getInitParameter(CONFIG_LOCATION_PARAM);
	if (configLocationParam != null) {
		wac.setConfigLocation(configLocationParam);
	}

	// The wac environment's #initPropertySources will be called in any case when the context
	// is refreshed; do it eagerly here to ensure servlet property sources are in place for
	// use in any post-processing or initialization that occurs below prior to #refresh
	ConfigurableEnvironment env = wac.getEnvironment();
	if (env instanceof ConfigurableWebEnvironment) {
		((ConfigurableWebEnvironment) env).initPropertySources(sc, null);
	}

	customizeContext(sc, wac);
	wac.refresh();
}
```
##### 方法进来首先检查是否设置过id，如果没有设置过id，那么就设置容器id。
```java 
if (ObjectUtils.identityToString(wac).equals(wac.getId())) {
	// The application context id is still set to its original default value
	// -> assign a more useful id based on available information
	String idParam = sc.getInitParameter(CONTEXT_ID_PARAM);
	if (idParam != null) {
		wac.setId(idParam);
	}
	else {
		// Generate default id...
		wac.setId(ConfigurableWebApplicationContext.APPLICATION_CONTEXT_ID_PREFIX +
				ObjectUtils.getDisplayString(sc.getContextPath()));
	}
}
```
容器id优先从配置文web.xml中获取，如果配置文件配置了容器id，那个就使用配置的容器id，否则使用默认生成的容器id。
##### 然后是获取spring配置文件。配置文件也是我们在web.xml配置的context-param
```java
String configLocationParam = sc.getInitParameter(CONFIG_LOCATION_PARAM);
if (configLocationParam != null) {
	wac.setConfigLocation(configLocationParam);
}
```
```xml
<context-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>classpath*:META-INF/spring/*.xml</param-value>
</context-param>
```
Spring的环境配置，暂时不深入了解，后续在学习Spring的bean装配时在介绍。

##### 接下来是一个扩展点————定制上下文，在容器刷新之前，给用户一个机会，可以做一些事情。
```
customizeContext(sc, wac);
```
跟进去看看
```java
protected void customizeContext(ServletContext sc, ConfigurableWebApplicationContext wac) {
	List<Class<ApplicationContextInitializer<ConfigurableApplicationContext>>> initializerClasses =
			determineContextInitializerClasses(sc);

	for (Class<ApplicationContextInitializer<ConfigurableApplicationContext>> initializerClass : initializerClasses) {
		Class<?> initializerContextClass =
				GenericTypeResolver.resolveTypeArgument(initializerClass, ApplicationContextInitializer.class);
		if (initializerContextClass != null && !initializerContextClass.isInstance(wac)) {
			throw new ApplicationContextException(String.format(
					"Could not apply context initializer [%s] since its generic parameter [%s] " +
					"is not assignable from the type of application context used by this " +
					"context loader: [%s]", initializerClass.getName(), initializerContextClass.getName(),
					wac.getClass().getName()));
		}
		this.contextInitializers.add(BeanUtils.instantiateClass(initializerClass));
	}

	AnnotationAwareOrderComparator.sort(this.contextInitializers);
	for (ApplicationContextInitializer<ConfigurableApplicationContext> initializer : this.contextInitializers) {
		initializer.initialize(wac);
	}
}
```
怎么定制上下文呢？看代码可以知道，要实现ApplicationContextInitializer接口
```java
public interface ApplicationContextInitializer<C extends ConfigurableApplicationContext> {

	/**
	 * Initialize the given application context.
	 * @param applicationContext the application to configure
	 */
	void initialize(C applicationContext);

}
```
接口就一个初始化方法，传入上下文。实现该接口，自己操作上下文，完成定制。实现了ApplicationContextInitializer接口之后，怎么配置让其生效呢？我们先看看是怎么找实现该接口的类的，进入determineContextInitializerClasses看看。
```java
protected List<Class<ApplicationContextInitializer<ConfigurableApplicationContext>>>
			determineContextInitializerClasses(ServletContext servletContext) {

	List<Class<ApplicationContextInitializer<ConfigurableApplicationContext>>> classes =
			new ArrayList<Class<ApplicationContextInitializer<ConfigurableApplicationContext>>>();

	String globalClassNames = servletContext.getInitParameter(GLOBAL_INITIALIZER_CLASSES_PARAM);
	if (globalClassNames != null) {
		for (String className : StringUtils.tokenizeToStringArray(globalClassNames, INIT_PARAM_DELIMITERS)) {
			classes.add(loadInitializerClass(className));
		}
	}

	String localClassNames = servletContext.getInitParameter(CONTEXT_INITIALIZER_CLASSES_PARAM);
	if (localClassNames != null) {
		for (String className : StringUtils.tokenizeToStringArray(localClassNames, INIT_PARAM_DELIMITERS)) {
			classes.add(loadInitializerClass(className));
		}
	}

	return classes;
}
```
可以看到，这里是通过 servletContext.getInitParameter(GLOBAL_INITIALIZER_CLASSES_PARAM)和servletContext.getInitParameter(CONTEXT_INITIALIZER_CLASSES_PARAM) 获取类名，和Spring配置文件一样，也是在web.xml中配置的。   

##### 最后是刷新上下文。
这里的刷新上下文，其实就是Spring读取配置稳定，解析bean定义，注册BeanDefinition，实例化bean，完成注入的过程，在此暂不深入。
```java
wac.refresh();
```

## 3. DispatcherServlet分析
### 3.1继承关系
![]({{site.url}}/{{site.blog-img}}/{{page.img-path}}/4.png)
DispatcherServlet间接实现了Servlet接口。在初始化阶段，会调用Servlet的init()方法，这是Servlet的入口，接下来我们找到入口，从入口分析。

### 3.2 追根溯源 Servlet的init()
本次分析的目的是研究DispatcherServlet的上下文和ContextLoaderListener的上下文是如何初始化和关联起来的，其它逻辑不是本文重点，将会被忽略。    

在GenericServlet类中实现了Servlet接口的init方法
```java
public void init(ServletConfig config) throws ServletException {
    this.config = config;
    this.init();
}
```
GenericServlet的init(ServletConfig config)方法除了设置了config并没有干其它的逻辑，而是调用了自身的init()方法，将具体逻辑委托给自己的子类处理。我们接着找谁实现了init()方法。在HttpServletBean类中找到了init()方法，看一下代码
```java
public final void init() throws ServletException {
	if (logger.isDebugEnabled()) {
		logger.debug("Initializing servlet '" + getServletName() + "'");
	}

	// Set bean properties from init parameters.
	PropertyValues pvs = new ServletConfigPropertyValues(getServletConfig(), this.requiredProperties);
	if (!pvs.isEmpty()) {
		try {
			BeanWrapper bw = PropertyAccessorFactory.forBeanPropertyAccess(this);
			ResourceLoader resourceLoader = new ServletContextResourceLoader(getServletContext());
			bw.registerCustomEditor(Resource.class, new ResourceEditor(resourceLoader, getEnvironment()));
			initBeanWrapper(bw);
			bw.setPropertyValues(pvs, true);
		}
		catch (BeansException ex) {
			if (logger.isErrorEnabled()) {
				logger.error("Failed to set bean properties on servlet '" + getServletName() + "'", ex);
			}
			throw ex;
		}
	}

	// Let subclasses do whatever initialization they like.
	initServletBean();

	if (logger.isDebugEnabled()) {
		logger.debug("Servlet '" + getServletName() + "' configured successfully");
	}
}
```
HttpServletBean的init()方法也没有看到有处理上下文的逻辑，但是看了initServletBean()这个方法，我们继续跟代码。
在FrameworkServlet类中重写了initServletBean()方法。
```java
@Override
protected final void initServletBean() throws ServletException {
	getServletContext().log("Initializing Spring FrameworkServlet '" + getServletName() + "'");
	if (this.logger.isInfoEnabled()) {
		this.logger.info("FrameworkServlet '" + getServletName() + "': initialization started");
	}
	long startTime = System.currentTimeMillis();

	try {
		this.webApplicationContext = initWebApplicationContext();
		initFrameworkServlet();
	}
	catch (ServletException ex) {
		this.logger.error("Context initialization failed", ex);
		throw ex;
	}
	catch (RuntimeException ex) {
		this.logger.error("Context initialization failed", ex);
		throw ex;
	}

	if (this.logger.isInfoEnabled()) {
		long elapsedTime = System.currentTimeMillis() - startTime;
		this.logger.info("FrameworkServlet '" + getServletName() + "': initialization completed in " +
				elapsedTime + " ms");
	}
}
```
在这里终于看到了与上下文有关的逻辑————initWebApplicationContext()方法，跟之。
```java
protected WebApplicationContext initWebApplicationContext() {
	WebApplicationContext rootContext =
			WebApplicationContextUtils.getWebApplicationContext(getServletContext());
	WebApplicationContext wac = null;

	if (this.webApplicationContext != null) {
		// A context instance was injected at construction time -> use it
		wac = this.webApplicationContext;
		if (wac instanceof ConfigurableWebApplicationContext) {
			ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) wac;
			if (!cwac.isActive()) {
				// The context has not yet been refreshed -> provide services such as
				// setting the parent context, setting the application context id, etc
				if (cwac.getParent() == null) {
					// The context instance was injected without an explicit parent -> set
					// the root application context (if any; may be null) as the parent
					cwac.setParent(rootContext);
				}
				configureAndRefreshWebApplicationContext(cwac);
			}
		}
	}
	if (wac == null) {
		// No context instance was injected at construction time -> see if one
		// has been registered in the servlet context. If one exists, it is assumed
		// that the parent context (if any) has already been set and that the
		// user has performed any initialization such as setting the context id
		wac = findWebApplicationContext();
	}
	if (wac == null) {
		// No context instance is defined for this servlet -> create a local one
		wac = createWebApplicationContext(rootContext);
	}

	if (!this.refreshEventReceived) {
		// Either the context is not a ConfigurableApplicationContext with refresh
		// support or the context injected at construction time had already been
		// refreshed -> trigger initial onRefresh manually here.
		onRefresh(wac);
	}

	if (this.publishContext) {
		// Publish the context as a servlet context attribute.
		String attrName = getServletContextAttributeName();
		getServletContext().setAttribute(attrName, wac);
		if (this.logger.isDebugEnabled()) {
			this.logger.debug("Published WebApplicationContext of servlet '" + getServletName() +
					"' as ServletContext attribute with name [" + attrName + "]");
		}
	}

	return wac;
}
```
我们看我们感兴趣的逻辑。
```
WebApplicationContext rootContext =
			WebApplicationContextUtils.getWebApplicationContext(getServletContext());
WebApplicationContext wac = null;
.
.
.
if (wac == null) {
	// No context instance is defined for this servlet -> create a local one
	wac = createWebApplicationContext(rootContext);
}
```
这里获取根上下文rootContext，并将rootContext作为参数，创建WebApplicationContext。这里获取的rootContext就是在ContextLoaderListener设置的WebApplicationContext。Talk is cheap. Show me the code.
```java
public static WebApplicationContext getWebApplicationContext(ServletContext sc) {
	return getWebApplicationContext(sc, WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE);
}
```
看到 WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE，我们回忆一下ContextLoaderListener是怎么设置上下文到ServletContext的。
```java
servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, this.context);
```
证据确凿。   
再来看createWebApplicationContext(rootContext)
```java
protected WebApplicationContext createWebApplicationContext(WebApplicationContext parent) {
	return createWebApplicationContext((ApplicationContext) parent);
}
```
```java
protected WebApplicationContext createWebApplicationContext(ApplicationContext parent) {
	Class<?> contextClass = getContextClass();
	if (this.logger.isDebugEnabled()) {
		this.logger.debug("Servlet with name '" + getServletName() +
				"' will try to create custom WebApplicationContext context of class '" +
				contextClass.getName() + "'" + ", using parent context [" + parent + "]");
	}
	if (!ConfigurableWebApplicationContext.class.isAssignableFrom(contextClass)) {
		throw new ApplicationContextException(
				"Fatal initialization error in servlet with name '" + getServletName() +
				"': custom WebApplicationContext class [" + contextClass.getName() +
				"] is not of type ConfigurableWebApplicationContext");
	}
	ConfigurableWebApplicationContext wac =
			(ConfigurableWebApplicationContext) BeanUtils.instantiateClass(contextClass);

	wac.setEnvironment(getEnvironment());
	wac.setParent(parent);
	wac.setConfigLocation(getContextConfigLocation());

	configureAndRefreshWebApplicationContext(wac);

	return wac;
}
```
看到createWebApplicationContext方法的代码，是不是似曾相识的感觉？这和ContextLoaderListener的 initWebApplicationContext方法的逻辑是不是很相似？
```java
wac.setParent(parent);
```
我们将目光放在上面一行代码上。这行代码将ContextLoaderListener的创建的WebApplicationContext和DispatcherServlet创建的WebApplicationContext关联了起来，将Web应用上下文的层次体现出来，我们也知道WebApplicationContext其实也是Spring的BeanFactory，所以这里也是设置了Spring Ioc容器的父子关系。最后就是configureAndRefreshWebApplicationContext了，这里就不分析了。




