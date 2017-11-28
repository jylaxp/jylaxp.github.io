---
layout: post
title: Spring 之 BeanDefinition解析
category: Spring
img-path: 2017-11-27-spring-BeanDefinition-parse
---

上篇分析了BeanDefinition的属性和使用场景，这篇我们分析BeanDefinition是如何解析和注册的。我们最常用的bean定义是通过xml和注解的方式，所以这里就只分析这两种方式的bean解析。
### xml方式
我们从XmlWebApplicationContext这个类入手分析，为什么是这个类，请参考[Spring 初始化 ContextLoaderListener 与 DispatcherServlet](https://jylaxp.github.io/spring/2017/11/22/ContextLoaderListener-And-DispatcherServlet.html) 

XmlWebApplicationContext类的继承关系
![]({{site.url}}/{{site.blog-img}}/{{page.img-path}}/XmlWebApplicationContext.png)


入口在AbstractApplicationContext类的refresh()方法。
```java
@Override
public void refresh() throws BeansException, IllegalStateException {
	synchronized (this.startupShutdownMonitor) {
		// Prepare this context for refreshing.
		prepareRefresh();

		// Tell the subclass to refresh the internal bean factory.
		ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

		// Prepare the bean factory for use in this context.
		prepareBeanFactory(beanFactory);

		try {
			// Allows post-processing of the bean factory in context subclasses.
			postProcessBeanFactory(beanFactory);

			// Invoke factory processors registered as beans in the context.
			invokeBeanFactoryPostProcessors(beanFactory);

			// Register bean processors that intercept bean creation.
			registerBeanPostProcessors(beanFactory);

			// Initialize message source for this context.
			initMessageSource();

			// Initialize event multicaster for this context.
			initApplicationEventMulticaster();

			// Initialize other special beans in specific context subclasses.
			onRefresh();

			// Check for listener beans and register them.
			registerListeners();

			// Instantiate all remaining (non-lazy-init) singletons.
			finishBeanFactoryInitialization(beanFactory);

			// Last step: publish corresponding event.
			finishRefresh();
		}

		catch (BeansException ex) {
			if (logger.isWarnEnabled()) {
				logger.warn("Exception encountered during context initialization - " +
						"cancelling refresh attempt: " + ex);
			}

			// Destroy already created singletons to avoid dangling resources.
			destroyBeans();

			// Reset 'active' flag.
			cancelRefresh(ex);

			// Propagate exception to caller.
			throw ex;
		}

		finally {
			// Reset common introspection caches in Spring's core, since we
			// might not ever need metadata for singleton beans anymore...
			resetCommonCaches();
		}
	}
}
```
跟进obtainFreshBeanFactory()方法，该方法获取beanFactory，获取到的beanFactory已经是完成了BeanDefinition的注册。

```java
/**
 * Tell the subclass to refresh the internal bean factory.
 * @return the fresh BeanFactory instance
 * @see #refreshBeanFactory()
 * @see #getBeanFactory()
 */
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
	refreshBeanFactory();
	ConfigurableListableBeanFactory beanFactory = getBeanFactory();
	if (logger.isDebugEnabled()) {
		logger.debug("Bean factory for " + getDisplayName() + ": " + beanFactory);
	}
	return beanFactory;
}
```

跟进refreshBeanFactory()方法，该方法的在AbstractRefreshableApplicationContext类中实现

```java
/**
 * This implementation performs an actual refresh of this context's underlying
 * bean factory, shutting down the previous bean factory (if any) and
 * initializing a fresh bean factory for the next phase of the context's lifecycle.
 */
@Override
protected final void refreshBeanFactory() throws BeansException {
	if (hasBeanFactory()) {
		destroyBeans();
		closeBeanFactory();
	}
	try {
		DefaultListableBeanFactory beanFactory = createBeanFactory();
		beanFactory.setSerializationId(getId());
		customizeBeanFactory(beanFactory);
		loadBeanDefinitions(beanFactory);
		synchronized (this.beanFactoryMonitor) {
			this.beanFactory = beanFactory;
		}
	}
	catch (IOException ex) {
		throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
	}
}
```

我们看到这里调用了createBeanFactory()创建了beanFactory，下面调用loadBeanDefinitions(beanFactory)进行加载bean定义。loadBeanDefinitions()方法的实现在XmlWebApplicationContext类提供。
```java
/**
 * Loads the bean definitions via an XmlBeanDefinitionReader.
 * @see org.springframework.beans.factory.xml.XmlBeanDefinitionReader
 * @see #initBeanDefinitionReader
 * @see #loadBeanDefinitions
 */
@Override
protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
	// Create a new XmlBeanDefinitionReader for the given BeanFactory.
	XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);

	// Configure the bean definition reader with this context's
	// resource loading environment.
	beanDefinitionReader.setEnvironment(getEnvironment());
	beanDefinitionReader.setResourceLoader(this);
	beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));

	// Allow a subclass to provide custom initialization of the reader,
	// then proceed with actually loading the bean definitions.
	initBeanDefinitionReader(beanDefinitionReader);
	loadBeanDefinitions(beanDefinitionReader);
}
```
到这里，才是真正的开始和beanDefinition解析相关了，前面的代码都是引子。
```java
// Create a new XmlBeanDefinitionReader for the given BeanFactory.
XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);
```
首先，创建了一个XmlBeanDefinitionReader，看名字就只可以知道，这个XmlBeanDefinitionReader要干的事情就是就是从xml读取BeanDefinition。XmlBeanDefinitionReader的构造方法的参数这里参入的beanFactory，我们臆测，XmlBeanDefinitionReader读取的BeanDefinition就是注册到这个beanFacctory容器里的。看一下XmlBeanDefinitionReader的构方法，参数类型是BeanDefinitionRegistr，这个是一个接口，提供BeanDefinition注册功能，可以看一下DefaultListableBeanFactory类的关系，可以看到DefaultListableBeanFactory实现了BeanDefinitionRegistry接口。
```java
/**
 * Create new XmlBeanDefinitionReader for the given bean factory.
 * @param registry the BeanFactory to load bean definitions into,
 * in the form of a BeanDefinitionRegistry
 */
public XmlBeanDefinitionReader(BeanDefinitionRegistry registry) {
	super(registry);
}
```

```java
// Configure the bean definition reader with this context's
// resource loading environment.
beanDefinitionReader.setEnvironment(getEnvironment());
beanDefinitionReader.setResourceLoader(this);
beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));
```
接着，配置BeanDefinitionReader上下文使用的环境，资源加载器等，**这里的资源加载器设置的this, 也就是XmlWebApplicationContext**。
```java
// Allow a subclass to provide custom initialization of the reader,
// then proceed with actually loading the bean definitions.
initBeanDefinitionReader(beanDefinitionReader);
loadBeanDefinitions(beanDefinitionReader);
```
然后是初始化BeanDefinitionReader，initBeanDefinitionReader(beanDefinitionReader)，这个是一个空方法，其实这是扩展点，留给子类实现，来对beanDefinitionReader进行一些操作。下面的loadBeanDefinitions(beanDefinitionReader)这个行代码是重点，跟进去一探究竟。
```java
protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws IOException {
	String[] configLocations = getConfigLocations();
	if (configLocations != null) {
		for (String configLocation : configLocations) {
			reader.loadBeanDefinitions(configLocation);
		}
	}
}
```
调用getConfigLocations()方法定位配置文件信息，它获取的其实就是在web.xml配置的contextConfigLocation的param-value。
```xml
<context-param>
	<param-name>contextConfigLocation</param-name>
	<param-value>classpath*:META-INF/spring/*.xml</param-value>
</context-param>
```
为什么是String[], 应为param-value节点的值可以是多个啊，我可以配置这样
```xml
<context-param>
	<param-name>contextConfigLocation</param-name>
	<param-value>classpath*:META-INF/spring/1.xml classpath*:META-INF/spring/2.xml</param-value>
</context-param>
```
代码接着往下看，reader.loadBeanDefinitions(configLocation)，控制权交给了XmlBeanDefinitionReader，我们分析这个类。先看一下类的关系。

![]({{site.url}}/{{site.blog-img}}/{{page.img-path}}/XmlBeanDefinitionReader.png)

我们跟一下reader.loadBeanDefinitions(configLocation)这方法，这个方法在AbstractBeanDefinitionReader类中
```java

@Override
public int loadBeanDefinitions(String location) throws BeanDefinitionStoreException {
	return loadBeanDefinitions(location, null);
}

public int loadBeanDefinitions(String location, Set<Resource> actualResources) throws BeanDefinitionStoreException {
	ResourceLoader resourceLoader = getResourceLoader();
	if (resourceLoader == null) {
		throw new BeanDefinitionStoreException(
				"Cannot import bean definitions from location [" + location + "]: no ResourceLoader available");
	}

	if (resourceLoader instanceof ResourcePatternResolver) {
		// Resource pattern matching available.
		try {
			Resource[] resources = ((ResourcePatternResolver) resourceLoader).getResources(location);
			int loadCount = loadBeanDefinitions(resources);
			if (actualResources != null) {
				for (Resource resource : resources) {
					actualResources.add(resource);
				}
			}
			if (logger.isDebugEnabled()) {
				logger.debug("Loaded " + loadCount + " bean definitions from location pattern [" + location + "]");
			}
			return loadCount;
		}
		catch (IOException ex) {
			throw new BeanDefinitionStoreException(
					"Could not resolve bean definition resource pattern [" + location + "]", ex);
		}
	}
	else {
		// Can only load single resources by absolute URL.
		Resource resource = resourceLoader.getResource(location);
		int loadCount = loadBeanDefinitions(resource);
		if (actualResources != null) {
			actualResources.add(resource);
		}
		if (logger.isDebugEnabled()) {
			logger.debug("Loaded " + loadCount + " bean definitions from location [" + location + "]");
		}
		return loadCount;
	}
}

@Override
public int loadBeanDefinitions(Resource... resources) throws BeanDefinitionStoreException {
	Assert.notNull(resources, "Resource array must not be null");
	int counter = 0;
	for (Resource resource : resources) {
		counter += loadBeanDefinitions(resource);
	}
	return counter;
}
```
第一个loadBeanDefinitions没什么可说的，我们看第二个loadBeanDefinitions。
```java
ResourceLoader resourceLoader = getResourceLoader();
```
这行代码获取的就是之前设置的资源加载器------XmlWebApplicationContext，通过它的类关系图，可以知道它间接实现了ResourcePatternResolver接口，所以会走下面的if分支。
```java
if (resourceLoader instanceof ResourcePatternResolver) {
	// Resource pattern matching available.
	try {
		Resource[] resources = ((ResourcePatternResolver) resourceLoader).getResources(location);
		int loadCount = loadBeanDefinitions(resources);
		if (actualResources != null) {
			for (Resource resource : resources) {
				actualResources.add(resource);
			}
		}
		if (logger.isDebugEnabled()) {
			logger.debug("Loaded " + loadCount + " bean definitions from location pattern [" + location + "]");
		}
		return loadCount;
	}
	catch (IOException ex) {
		throw new BeanDefinitionStoreException(
				"Could not resolve bean definition resource pattern [" + location + "]", ex);
	}
}
```
```java
Resource[] resources = ((ResourcePatternResolver) resourceLoader).getResources(location);
```
这行代码，完成了通配符的匹配，资源文件的定位。
```java
int loadCount = loadBeanDefinitions(resources);
```
这个行代码就是调用的第三个loadBeanDefinitions，第三个loadBeanDefinitions又调用了loadBeanDefinitions，真是"这里的山路十八弯, 这里的水路九连环"啊，命令一层一层的下达，到现在还没有找到干活的人，全是协调组织者，真像是国家的行政机关啊，全是开证明盖章的。跟进去吧～，这个方法在XmlBeanDefinitionReader类中。
```java
@Override
public int loadBeanDefinitions(Resource resource) throws BeanDefinitionStoreException {
	return loadBeanDefinitions(new EncodedResource(resource));
}

public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
	Assert.notNull(encodedResource, "EncodedResource must not be null");
	if (logger.isInfoEnabled()) {
		logger.info("Loading XML bean definitions from " + encodedResource.getResource());
	}

	Set<EncodedResource> currentResources = this.resourcesCurrentlyBeingLoaded.get();
	if (currentResources == null) {
		currentResources = new HashSet<EncodedResource>(4);
		this.resourcesCurrentlyBeingLoaded.set(currentResources);
	}
	if (!currentResources.add(encodedResource)) {
		throw new BeanDefinitionStoreException(
				"Detected cyclic loading of " + encodedResource + " - check your import definitions!");
	}
	try {
		InputStream inputStream = encodedResource.getResource().getInputStream();
		try {
			InputSource inputSource = new InputSource(inputStream);
			if (encodedResource.getEncoding() != null) {
				inputSource.setEncoding(encodedResource.getEncoding());
			}
			return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
		}
		finally {
			inputStream.close();
		}
	}
	catch (IOException ex) {
		throw new BeanDefinitionStoreException(
				"IOException parsing XML document from " + encodedResource.getResource(), ex);
	}
	finally {
		currentResources.remove(encodedResource);
		if (currentResources.isEmpty()) {
			this.resourcesCurrentlyBeingLoaded.remove();
		}
	}
}
```
啪啪，又盖了两个章。逻辑很简单，就是获取InputStream输入流，也就是打开文件，然后调doLoadBeanDefinitions(inputSource, encodedResource.getResource())干活，不容易啊，终于找到干活的了。
```java
protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
		throws BeanDefinitionStoreException {
	try {
		Document doc = doLoadDocument(inputSource, resource);
		return registerBeanDefinitions(doc, resource);
	}
	catch (BeanDefinitionStoreException ex) {
		throw ex;
	}
	catch (SAXParseException ex) {
		throw new XmlBeanDefinitionStoreException(resource.getDescription(),
				"Line " + ex.getLineNumber() + " in XML document from " + resource + " is invalid", ex);
	}
	catch (SAXException ex) {
		throw new XmlBeanDefinitionStoreException(resource.getDescription(),
				"XML document from " + resource + " is invalid", ex);
	}
	catch (ParserConfigurationException ex) {
		throw new BeanDefinitionStoreException(resource.getDescription(),
				"Parser configuration exception parsing XML from " + resource, ex);
	}
	catch (IOException ex) {
		throw new BeanDefinitionStoreException(resource.getDescription(),
				"IOException parsing XML document from " + resource, ex);
	}
	catch (Throwable ex) {
		throw new BeanDefinitionStoreException(resource.getDescription(),
				"Unexpected exception parsing XML document from " + resource, ex);
	}
}
```
关键代码就两行
```java
Document doc = doLoadDocument(inputSource, resource);
```
读取资源xml文件，验证xml文件，解析xml成Document。
```java
registerBeanDefinitions(doc, resource);
```
注册BeanDefinition。
```java
public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
	BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
	int countBefore = getRegistry().getBeanDefinitionCount();
	documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
	return getRegistry().getBeanDefinitionCount() - countBefore;
}
```
这里创建了BeanDefinitionDocumentReader对象来解析注册BeanDefinition，有一点要注意
```java
documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
```
这个行代码里的createReaderContext(resource)，它是获取一个XmlReaderContext对象，解析BeanDefinition要用到的上下文，它为自定义spring schemas提供了扩展，以注解定义的BeanDefinition的解析也是通过这种方式做的扩展，这里不深入分析，在后面分析解析以注解方式定义的bean时在深入的研究。



