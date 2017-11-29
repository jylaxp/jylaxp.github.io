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
首先，创建了一个XmlBeanDefinitionReader，看名字就只可以知道，这个XmlBeanDefinitionReader要干的事情就是就是从xml读取BeanDefinition。XmlBeanDefinitionReader的构造方法的参数这里参入的beanFactory，我们臆测，XmlBeanDefinitionReader读取的BeanDefinition就是注册到这个beanFacctory容器里的。看一下XmlBeanDefinitionReader的构方法，参数类型是BeanDefinitionRegistr，这个是一个接口，提供BeanDefinition注册功能，可以看一下DefaultListableBeanFactory类的关系，可以看到DefaultListableBeanFactory实现了BeanDefinitionRegistry接口，DefaultListableBeanFactory本身也是一个Registry。
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

```java
protected void doRegisterBeanDefinitions(Element root) {
	// Any nested <beans> elements will cause recursion in this method. In
	// order to propagate and preserve <beans> default-* attributes correctly,
	// keep track of the current (parent) delegate, which may be null. Create
	// the new (child) delegate with a reference to the parent for fallback purposes,
	// then ultimately reset this.delegate back to its original (parent) reference.
	// this behavior emulates a stack of delegates without actually necessitating one.
	BeanDefinitionParserDelegate parent = this.delegate;
	this.delegate = createDelegate(getReaderContext(), root, parent);

	if (this.delegate.isDefaultNamespace(root)) {
		String profileSpec = root.getAttribute(PROFILE_ATTRIBUTE);
		if (StringUtils.hasText(profileSpec)) {
			String[] specifiedProfiles = StringUtils.tokenizeToStringArray(
					profileSpec, BeanDefinitionParserDelegate.MULTI_VALUE_ATTRIBUTE_DELIMITERS);
			if (!getReaderContext().getEnvironment().acceptsProfiles(specifiedProfiles)) {
				if (logger.isInfoEnabled()) {
					logger.info("Skipped XML bean definition file due to specified profiles [" + profileSpec +
							"] not matching: " + getReaderContext().getResource());
				}
				return;
			}
		}
	}

	preProcessXml(root);
	parseBeanDefinitions(root, this.delegate);
	postProcessXml(root);

	this.delegate = parent;
}
```
上面那堆注释是来解析第一行代码和最后一行代码的，解释为什么要这样做。这里解释一下，这是什么场景。该方法可以看做是对```<beans/>```节点的解析,如果```<beans/>```节点嵌套了```<beans/>```节点，则该方法会递归调用，但是每个```<beans/>```节点都有它自己的默认值，默认值的解析和暂存都是在BeanDefinitionParserDelegate这个委托中的，也就是说这个委托关联了```<beans/>```的默认值，当```<beans/>```发生嵌套，递归调用此方法，如果不是重新创建一个委托，而是使用原来的委托，则会改变该委托暂存的默认值，导致解析错误。看一下```createDelegate(getReaderContext(), root, parent)```的代码
```java
protected BeanDefinitionParserDelegate createDelegate(
		XmlReaderContext readerContext, Element root, BeanDefinitionParserDelegate parentDelegate) {

	BeanDefinitionParserDelegate delegate = new BeanDefinitionParserDelegate(readerContext);
	delegate.initDefaults(root, parentDelegate);
	return delegate;
}
```
可以看到，这个创建了一个新的BeanDefinitionParserDelegate对象，并调用它的```initDefaults```方法初始化了默认值。
```java
if (this.delegate.isDefaultNamespace(root)) {
	String profileSpec = root.getAttribute(PROFILE_ATTRIBUTE);
	if (StringUtils.hasText(profileSpec)) {
		String[] specifiedProfiles = StringUtils.tokenizeToStringArray(
				profileSpec, BeanDefinitionParserDelegate.MULTI_VALUE_ATTRIBUTE_DELIMITERS);
		if (!getReaderContext().getEnvironment().acceptsProfiles(specifiedProfiles)) {
			if (logger.isInfoEnabled()) {
				logger.info("Skipped XML bean definition file due to specified profiles [" + profileSpec +
						"] not matching: " + getReaderContext().getResource());
			}
			return;
		}
	}
}
```
这个是和profile有关，暂不研究。
```java
preProcessXml(root);
parseBeanDefinitions(root, this.delegate);
postProcessXml(root);
```
preProcessXml(root)和postProcessXml(root)都是空实现，留给子类扩展，parseBeanDefinitions(root, this.delegate)才是我们翻山越岭要找的真相。
```java
protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
	if (delegate.isDefaultNamespace(root)) {
		NodeList nl = root.getChildNodes();
		for (int i = 0; i < nl.getLength(); i++) {
			Node node = nl.item(i);
			if (node instanceof Element) {
				Element ele = (Element) node;
				if (delegate.isDefaultNamespace(ele)) {
					parseDefaultElement(ele, delegate);
				}
				else {
					delegate.parseCustomElement(ele);
				}
			}
		}
	}
	else {
		delegate.parseCustomElement(root);
	}
}
```
先判断root节点是否是默认的命名空间```http://www.springframework.org/schema/beans```，如果是，这个按照默认提供的解析逻辑解析该节点，否则是通过spring schemas提供了扩展，则按照扩展的标签的逻辑解析，注解定义的bean也是通过这种方式提供的解析处理逻辑，后面分析注解定义的bean扫描和自定扩展标签的时候就从这个地方入手，前面的逻辑就不重复了。 我们先分析默认元素的解析。
```java
private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
	if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) {
		importBeanDefinitionResource(ele);
	}
	else if (delegate.nodeNameEquals(ele, ALIAS_ELEMENT)) {
		processAliasRegistration(ele);
	}
	else if (delegate.nodeNameEquals(ele, BEAN_ELEMENT)) {
		processBeanDefinition(ele, delegate);
	}
	else if (delegate.nodeNameEquals(ele, NESTED_BEANS_ELEMENT)) {
		// recurse
		doRegisterBeanDefinitions(ele);
	}
}
```
这里处理了４种情况:

- 处理```<import/>```标签。```<import/>```是导入另外一个spring配置文件，处理逻辑和上面资源定位，读取配置文件，然后解析出BeanDefinition逻辑一样。
- 处理```<alias/>```标签。解析别名，这个逻辑比较简单，只要在registry中注册别名和beanName的对应关系即可。
- 处理```<bean/>```标签。这个就是解析BeanDefinition了，我们分析的重点。
- 处理```<beans/>```标签。处理嵌套的处理```<bean/>```标签，可以看到这里递归调用了doRegisterBeanDefinitions方法。
```java
protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
	BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
	if (bdHolder != null) {
		bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
		try {
			// Register the final decorated instance.
			BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
		}
		catch (BeanDefinitionStoreException ex) {
			getReaderContext().error("Failed to register bean definition with name '" +
					bdHolder.getBeanName() + "'", ele, ex);
		}
		// Send registration event.
		getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
	}
}
```
processBeanDefinition主要完成了三件事情： 

- 使用BeanDefinitionParserDelegate解析出BeanDefinition，但这里返回处理的是BeanDefinitionHolder，为什么是BeanDefinitionHolder，看一下这个类属性，不做解释了。
```java
public class BeanDefinitionHolder implements BeanMetadataElement {

	private final BeanDefinition beanDefinition;

	private final String beanName;

	private final String[] aliases;
}
```
- 装饰解析出来的BeanDefinitionHolder，其实就是装饰BeanDefinition。
- 向registry注册BeanDefinition。
我们来看```bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);```这里做了什么。看代码里加的注释吧，不想啰嗦了。

```java
/**
 * Parses the supplied {@code <bean>} element. May return {@code null}
 * if there were errors during parse. Errors are reported to the
 * {@link org.springframework.beans.factory.parsing.ProblemReporter}.
 */
public BeanDefinitionHolder parseBeanDefinitionElement(Element ele) {
	return parseBeanDefinitionElement(ele, null);
}

/**
 * Parses the supplied {@code <bean>} element. May return {@code null}
 * if there were errors during parse. Errors are reported to the
 * {@link org.springframework.beans.factory.parsing.ProblemReporter}.
 */
public BeanDefinitionHolder parseBeanDefinitionElement(Element ele, BeanDefinition containingBean) {
	// 获取<bean/>标签的id属性(beanName)
	String id = ele.getAttribute(ID_ATTRIBUTE);
	
	// 获取<bean/>标签的name属性(别名)
	String nameAttr = ele.getAttribute(NAME_ATTRIBUTE);

	List<String> aliases = new ArrayList<String>();
	if (StringUtils.hasLength(nameAttr)) {
		// 别名拆分
		String[] nameArr = StringUtils.tokenizeToStringArray(nameAttr, MULTI_VALUE_ATTRIBUTE_DELIMITERS);
		aliases.addAll(Arrays.asList(nameArr));
	}

	String beanName = id;
	if (!StringUtils.hasText(beanName) && !aliases.isEmpty()) {
		// 如果没有配置beanNam，但是配置了别名，取第一个别名作为beanName
		beanName = aliases.remove(0);
		if (logger.isDebugEnabled()) {
			logger.debug("No XML 'id' specified - using '" + beanName +
					"' as bean name and " + aliases + " as aliases");
		}
	}

	if (containingBean == null) {
		// 检查beanName和别名的唯一性。
		// 这里的唯一性是在同一容器中唯一而不是一个应用中唯一，在不同的容器中可以重复，比如有父子关系的两个容器，可以有相同beanName和别名的bean
		checkNameUniqueness(beanName, aliases, ele);
	}

	// 解析beanDefinition
	AbstractBeanDefinition beanDefinition = parseBeanDefinitionElement(ele, beanName, containingBean);
	if (beanDefinition != null) {
		// 如果还是没有beanName，则自动生成beanName
		if (!StringUtils.hasText(beanName)) {
			try {
				if (containingBean != null) {
					beanName = BeanDefinitionReaderUtils.generateBeanName(
							beanDefinition, this.readerContext.getRegistry(), true);
				}
				else {
					beanName = this.readerContext.generateBeanName(beanDefinition);
					// Register an alias for the plain bean class name, if still possible,
					// if the generator returned the class name plus a suffix.
					// This is expected for Spring 1.2/2.0 backwards compatibility.
					String beanClassName = beanDefinition.getBeanClassName();
					if (beanClassName != null &&
							beanName.startsWith(beanClassName) && beanName.length() > beanClassName.length() &&
							!this.readerContext.getRegistry().isBeanNameInUse(beanClassName)) {
						aliases.add(beanClassName);
					}
				}
				if (logger.isDebugEnabled()) {
					logger.debug("Neither XML 'id' nor 'name' specified - " +
							"using generated bean name [" + beanName + "]");
				}
			}
			catch (Exception ex) {
				error(ex.getMessage(), ele);
				return null;
			}
		}
		String[] aliasesArray = StringUtils.toStringArray(aliases);
		return new BeanDefinitionHolder(beanDefinition, beanName, aliasesArray);
	}

	return null;
}
```
接着看beanDefinition解析parseBeanDefinitionElement
```java
public AbstractBeanDefinition parseBeanDefinitionElement(
		Element ele, String beanName, BeanDefinition containingBean) {

	this.parseState.push(new BeanEntry(beanName));

	String className = null;
	// 解析class属性
	if (ele.hasAttribute(CLASS_ATTRIBUTE)) {
		className = ele.getAttribute(CLASS_ATTRIBUTE).trim();
	}

	try {
		String parent = null;
		// 解析parent属性
		if (ele.hasAttribute(PARENT_ATTRIBUTE)) {
			parent = ele.getAttribute(PARENT_ATTRIBUTE);
		}
		
		// 创建BeanDefinition
		AbstractBeanDefinition bd = createBeanDefinition(className, parent);

		// 解析bean的其他属性
		parseBeanDefinitionAttributes(ele, beanName, containingBean, bd);
		
		// bean的描述
		bd.setDescription(DomUtils.getChildElementValueByTagName(ele, DESCRIPTION_ELEMENT));
		
		parseMetaElements(ele, bd);
		
		// override方法
		parseLookupOverrideSubElements(ele, bd.getMethodOverrides());
		
		// replace方法
		parseReplacedMethodSubElements(ele, bd.getMethodOverrides());

		// 构造方法参数
		parseConstructorArgElements(ele, bd);
		
		// setter注入的属性
		parsePropertyElements(ele, bd);
		parseQualifierElements(ele, bd);

		bd.setResource(this.readerContext.getResource());
		bd.setSource(extractSource(ele));

		return bd;
	}
	catch (ClassNotFoundException ex) {
		error("Bean class [" + className + "] not found", ele, ex);
	}
	catch (NoClassDefFoundError err) {
		error("Class that bean class [" + className + "] depends on not found", ele, err);
	}
	catch (Throwable ex) {
		error("Unexpected failure during bean definition parsing", ele, ex);
	}
	finally {
		this.parseState.pop();
	}

	return null;
}
```
可以看到，这段代码就是获取我们在xml定义的bean信息，转换成BeanDefinition对象。这个获取的BeanDefinition是AbstractBeanDefinition，那是哪个具体的BeanDefinition？在[Spring 之 BeanDefinition](https://jylaxp.github.io/2017/11/23/spring-BeanDefinition.html) 中给出的GenericBeanDefinition，我们验证一下。
```java
protected AbstractBeanDefinition createBeanDefinition(String className, String parentName)
		throws ClassNotFoundException {

	return BeanDefinitionReaderUtils.createBeanDefinition(
			parentName, className, this.readerContext.getBeanClassLoader());
}

// BeanDefinitionReaderUtils.createBeanDefinition
public static AbstractBeanDefinition createBeanDefinition(
		String parentName, String className, ClassLoader classLoader) throws ClassNotFoundException {

	GenericBeanDefinition bd = new GenericBeanDefinition();
	bd.setParentName(parentName);
	if (className != null) {
		if (classLoader != null) {
			bd.setBeanClass(ClassUtils.forName(className, classLoader));
		}
		else {
			bd.setBeanClassName(className);
		}
	}
	return bd;
}
```

解析xml标签和属性的过程很无聊的，无非就是调ele.getAttribute获取响应的属性的值，然后再调用bd.setXXX方法，给bd赋值，没什么可说的，标签解析完了，也就完成了bean从xml到BeanDefinition的转换了。不过，还有几点值得说说的。

- 解析```<bean/>```标签的属性。**如果没有指定属性，则使用默认值。**
```java
// 解析bean的其他属性
parseBeanDefinitionAttributes(ele, beanName, containingBean, bd);
```

- 解析构造方法的参数和setter方法的参数。

```java
// 构造方法参数
parseConstructorArgElements(ele, bd);

// setter注入的属性
parsePropertyElements(ele, bd);
```
构造参数和setter方法的参数比较复杂:
1. 基本类型
2. 其他的bean
3. 集合类型(Array, List, Set, Map )
4. Property
针对这些类型的参数，做了不同的解析

```java
public Object parsePropertySubElement(Element ele, BeanDefinition bd) {
	return parsePropertySubElement(ele, bd, null);
}

/**
 * Parse a value, ref or collection sub-element of a property or
 * constructor-arg element.
 * @param ele subelement of property element; we don't know which yet
 * @param defaultValueType the default type (class name) for any
 * {@code <value>} tag that might be created
 */
public Object parsePropertySubElement(Element ele, BeanDefinition bd, String defaultValueType) {
	if (!isDefaultNamespace(ele)) {
		// 子定义解析
		return parseNestedCustomElement(ele, bd);
	}
	else if (nodeNameEquals(ele, BEAN_ELEMENT)) {
		// bean 节点，　递归解析
		BeanDefinitionHolder nestedBd = parseBeanDefinitionElement(ele, bd);
		if (nestedBd != null) {
			nestedBd = decorateBeanDefinitionIfRequired(ele, nestedBd, bd);
		}
		return nestedBd;
	}
	else if (nodeNameEquals(ele, REF_ELEMENT)) {
		// ref引用节点
		// A generic reference to any name of any bean.
		String refName = ele.getAttribute(BEAN_REF_ATTRIBUTE);
		boolean toParent = false;
		if (!StringUtils.hasLength(refName)) {
			// A reference to the id of another bean in the same XML file.
			refName = ele.getAttribute(LOCAL_REF_ATTRIBUTE);
			if (!StringUtils.hasLength(refName)) {
				// A reference to the id of another bean in a parent context.
				refName = ele.getAttribute(PARENT_REF_ATTRIBUTE);
				toParent = true;
				if (!StringUtils.hasLength(refName)) {
					error("'bean', 'local' or 'parent' is required for <ref> element", ele);
					return null;
				}
			}
		}
		if (!StringUtils.hasText(refName)) {
			error("<ref> element contains empty target attribute", ele);
			return null;
		}
		RuntimeBeanReference ref = new RuntimeBeanReference(refName, toParent);
		ref.setSource(extractSource(ele));
		return ref;
	}
	else if (nodeNameEquals(ele, IDREF_ELEMENT)) {
		// idref节点
		return parseIdRefElement(ele);
	}
	else if (nodeNameEquals(ele, VALUE_ELEMENT)) {
		// value节点
		return parseValueElement(ele, defaultValueType);
	}
	else if (nodeNameEquals(ele, NULL_ELEMENT)) {
		// It's a distinguished null value. Let's wrap it in a TypedStringValue
		// object in order to preserve the source location.
		TypedStringValue nullHolder = new TypedStringValue(null);
		nullHolder.setSource(extractSource(ele));
		return nullHolder;
	}
	else if (nodeNameEquals(ele, ARRAY_ELEMENT)) {
		// Array节点
		return parseArrayElement(ele, bd);
	}
	else if (nodeNameEquals(ele, LIST_ELEMENT)) {
		// List 节点
		return parseListElement(ele, bd);
	}
	else if (nodeNameEquals(ele, SET_ELEMENT)) {
		// Set 节点
		return parseSetElement(ele, bd);
	}
	else if (nodeNameEquals(ele, MAP_ELEMENT)) {
		// map 节点
		return parseMapElement(ele, bd);
	}
	else if (nodeNameEquals(ele, PROPS_ELEMENT)) {
		return parsePropsElement(ele);
	}
	else {
		error("Unknown property sub-element: [" + ele.getNodeName() + "]", ele);
		return null;
	}
}
```
Array, List, Set 节点代码逻辑一样，值看Array就可以了。

```java
public Object parseArrayElement(Element arrayEle, BeanDefinition bd) {
	// 获取元素的类型
	String elementType = arrayEle.getAttribute(VALUE_TYPE_ATTRIBUTE);
	NodeList nl = arrayEle.getChildNodes();
	ManagedArray target = new ManagedArray(elementType, nl.getLength());
	target.setSource(extractSource(arrayEle));
	target.setElementTypeName(elementType);
	target.setMergeEnabled(parseMergeAttribute(arrayEle));
	
	// 解析集合元素
	parseCollectionElements(nl, target, bd, elementType);
	return target;
}

protected void parseCollectionElements(
		NodeList elementNodes, Collection<Object> target, BeanDefinition bd, String defaultElementType) {

	for (int i = 0; i < elementNodes.getLength(); i++) {
		Node node = elementNodes.item(i);
		if (node instanceof Element && !nodeNameEquals(node, DESCRIPTION_ELEMENT)) {
			// 递归调用parsePropertySubElement解析节点
			target.add(parsePropertySubElement((Element) node, bd, defaultElementType));
		}
	}
}
```
Map解析的代码看着比较复杂
```java
public Map<Object, Object> parseMapElement(Element mapEle, BeanDefinition bd) {
	// 获取key的默认类型，　map节点的属性
	String defaultKeyType = mapEle.getAttribute(KEY_TYPE_ATTRIBUTE);
	
	// 获取value的默认类型，　map节点的属性
	String defaultValueType = mapEle.getAttribute(VALUE_TYPE_ATTRIBUTE);

	// 获取map节点的所有entry
	List<Element> entryEles = DomUtils.getChildElementsByTagName(mapEle, ENTRY_ELEMENT);
	ManagedMap<Object, Object> map = new ManagedMap<Object, Object>(entryEles.size());
	map.setSource(extractSource(mapEle));
	map.setKeyTypeName(defaultKeyType);
	map.setValueTypeName(defaultValueType);
	map.setMergeEnabled(parseMergeAttribute(mapEle));

	for (Element entryEle : entryEles) {
		// Should only have one value child element: ref, value, list, etc.
		// Optionally, there might be a key child element.
		// 检查entry节点的子节点的合法性，只能有一个key节点和一个value节点
		NodeList entrySubNodes = entryEle.getChildNodes();
		Element keyEle = null;
		Element valueEle = null;
		for (int j = 0; j < entrySubNodes.getLength(); j++) {
			Node node = entrySubNodes.item(j);
			if (node instanceof Element) {
				Element candidateEle = (Element) node;
				if (nodeNameEquals(candidateEle, KEY_ELEMENT)) {
					if (keyEle != null) {
						error("<entry> element is only allowed to contain one <key> sub-element", entryEle);
					}
					else {
						keyEle = candidateEle;
					}
				}
				else {
					// Child element is what we're looking for.
					if (nodeNameEquals(candidateEle, DESCRIPTION_ELEMENT)) {
						// the element is a <description> -> ignore it
					}
					else if (valueEle != null) {
						error("<entry> element must not contain more than one value sub-element", entryEle);
					}
					else {
						valueEle = candidateEle;
					}
				}
			}
		}

		// 解析出key, 　key属性，key-ref属性，　key节点不能同是存在
		// Extract key from attribute or sub-element.
		Object key = null;
		boolean hasKeyAttribute = entryEle.hasAttribute(KEY_ATTRIBUTE);
		boolean hasKeyRefAttribute = entryEle.hasAttribute(KEY_REF_ATTRIBUTE);
		if ((hasKeyAttribute && hasKeyRefAttribute) ||
				((hasKeyAttribute || hasKeyRefAttribute)) && keyEle != null) {
			error("<entry> element is only allowed to contain either " +
					"a 'key' attribute OR a 'key-ref' attribute OR a <key> sub-element", entryEle);
		}
		if (hasKeyAttribute) {
			// key属性
			key = buildTypedStringValueForMap(entryEle.getAttribute(KEY_ATTRIBUTE), defaultKeyType, entryEle);
		}
		else if (hasKeyRefAttribute) {
			// key-ref属性
			String refName = entryEle.getAttribute(KEY_REF_ATTRIBUTE);
			if (!StringUtils.hasText(refName)) {
				error("<entry> element contains empty 'key-ref' attribute", entryEle);
			}
			RuntimeBeanReference ref = new RuntimeBeanReference(refName);
			ref.setSource(extractSource(entryEle));
			key = ref;
		}
		else if (keyEle != null) {
			// key节点
			key = parseKeyElement(keyEle, bd, defaultKeyType);
		}
		else {
			error("<entry> element must specify a key", entryEle);
		}

		// 解析出value，　value属性，　value-ref属性，　value节点　不能同时存在
		// Extract value from attribute or sub-element.
		Object value = null;
		boolean hasValueAttribute = entryEle.hasAttribute(VALUE_ATTRIBUTE);
		boolean hasValueRefAttribute = entryEle.hasAttribute(VALUE_REF_ATTRIBUTE);
		boolean hasValueTypeAttribute = entryEle.hasAttribute(VALUE_TYPE_ATTRIBUTE);
		if ((hasValueAttribute && hasValueRefAttribute) ||
				((hasValueAttribute || hasValueRefAttribute)) && valueEle != null) {
			error("<entry> element is only allowed to contain either " +
					"'value' attribute OR 'value-ref' attribute OR <value> sub-element", entryEle);
		}
		if ((hasValueTypeAttribute && hasValueRefAttribute) ||
			(hasValueTypeAttribute && !hasValueAttribute) ||
				(hasValueTypeAttribute && valueEle != null)) {
			error("<entry> element is only allowed to contain a 'value-type' " +
					"attribute when it has a 'value' attribute", entryEle);
		}
		if (hasValueAttribute) {
			// value 属性
			String valueType = entryEle.getAttribute(VALUE_TYPE_ATTRIBUTE);
			if (!StringUtils.hasText(valueType)) {
				valueType = defaultValueType;
			}
			value = buildTypedStringValueForMap(entryEle.getAttribute(VALUE_ATTRIBUTE), valueType, entryEle);
		}
		else if (hasValueRefAttribute) {
			// value-ref属性
			String refName = entryEle.getAttribute(VALUE_REF_ATTRIBUTE);
			if (!StringUtils.hasText(refName)) {
				error("<entry> element contains empty 'value-ref' attribute", entryEle);
			}
			RuntimeBeanReference ref = new RuntimeBeanReference(refName);
			ref.setSource(extractSource(entryEle));
			value = ref;
		}
		else if (valueEle != null) {
			// value节点
			value = parsePropertySubElement(valueEle, bd, defaultValueType);
		}
		else {
			error("<entry> element must specify a value", entryEle);
		}

		// Add final key and value to the Map.
		map.put(key, value);
	}

	return map;
}
```
代码看似复杂，其实就是key和value定义三种情况，key和value可以是entry节点的属性，ref属性或子节点，分这三种情况进行解析的，上例子：

```xml
<bean class="java.util.HashMap">
	<constructor-arg>
	    <map key-type="java.lang.Object" value-type="java.lang.Object">
		<entry key="k1" value="v1"/>
		<entry key-ref="car" value="v1"/>
		<entry key="v3" value-ref="car"/>
		<entry>
		    <key>
		        <value>k2</value>
		    </key>
		    <value>v2</value>
		</entry>
		<entry>
		    <key>
		        <value>k3</value>
		    </key>
		    <ref bean="car"/>
		</entry>
	    </map>
	</constructor-arg>
</bean>
```
把这个例子带入解析代码看的话逻辑就很清晰了。
至此，基于xml配置，并且没有扩展标签的bean解析、注册分析完了。