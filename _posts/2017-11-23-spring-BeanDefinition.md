---
layout: post
title: Spring 之 BeanDefinition
---
# Spring 之 BeanDefinition

### 1. 使用场景
bean定义，用于表示bean的各种属性和bean与bean的关系，是 spring freamwork 描述bean定义的对象。
    在spring freamwork中，将xml描述的bean属性转换成BeanDefinition，作为后续实例化bean和自动装配的原数据。

### 2. 业务场景
在一个应用中，一个bean从定义到可以使用，我们来看看我们一般会对它做哪些操作。

- 定义一个类，定义类的属性，方法
- 类实例化，如果是有参构造方法，在实例化之前准备好构造方法需要的参数
- 类实例化实例化之后，可能要做一些初始化操作
- 初始化完成之后，就可以使用了。在使用结束之后，可能还要释放一些资源。

基于以上分析，我们如何使用对象描述这些信息，并使用一套“标准”流程完成这些操作，达到获取到这个bean就是可以使用的呢？
首先，我们得把这些信息抽象出来，使用一个对象表示它。抽象出来哪些信息呢？我们结合上面的过程来一步一步分析。 

- bean定义
	- 我是谁，bean的类型(class)
	- 我的父母是谁，bean的继承关系
	- 我有什么，类的属性
- bean实例化
	- 对于有参构造方法，每一个构造方法的参数类型、顺序也得描述出来
- 初始化操作
	- 告诉我你的初始化方法，不然我调用谁初始化
- bean销毁
	- 告诉我你的销毁方法，不然我调用谁释放资源
	
以上信息可以对bean做最基本的描述。我们知道，spring的扩展性很强，它在最基本的定义描述加了一些spring freamwork自身需要使用的属性和一些可以订制bean的扩展信息，接下来，我们来看BeanDefinition.

### 3. BeanDefinition
BeanDefinition是一个接口，定义了BeanDefinition的行为。AbstractBeanDefinition为BeanDefinition提供了默认实现，我们来看看它定义了哪些属性。
	
```java
// 类名
private volatile Object beanClass;

// 类的构造方法参数
private ConstructorArgumentValues constructorArgumentValues;

// 类属性, bean定义的property
private MutablePropertyValues propertyValues;

// 初始化方法
private String initMethodName;

// 销毁方法
private String destroyMethodName;

// 作用域
private String scope = SCOPE_DEFAULT;

// 是否是抽象bean
private boolean abstractFlag = false;

// 是否懒初始化。在spring高级容器AbstractApplicationContext中，会预先示例话非懒初始化的bean
private boolean lazyInit = false;

// 自动装配模式(按name装配还是按type装配)
private int autowireMode = AUTOWIRE_NO;

// 依赖检查
private int dependencyCheck = DEPENDENCY_CHECK_NONE;

// 依赖关系，在定义bean的时候，由depend-on属性设置，被依赖的bean在该bean之前被容器初始化
private String[] dependsOn;

// 该bean是否参数注入，　true:参与自动注入，可以注入到其他bean；false:不参数自动注入，不会注入其他bean中
private boolean autowireCandidate = true;

// 按类型注入是，当一个接口有多个实现时，告诉spring注入这个primary=true的bean，相当于消除歧义
private boolean primary = false;

// Qualifier
private final Map<String, AutowireCandidateQualifier> qualifiers =
		new LinkedHashMap<String, AutowireCandidateQualifier>(0);

// 是否允许访问非共有的属性(使用场景未研究)
private boolean nonPublicAccessAllowed = true;

// 调用构造函数时，是否采用宽松匹配(使用场景未研究)
private boolean lenientConstructorResolution = true;

// 跟Bean定义时的look-up有关, 方法覆盖，增强扩展bean
private MethodOverrides methodOverrides = new MethodOverrides();

// 工厂方法，实例化bean是调用此工厂方法获取bean示例
private String factoryMethodName;

// bean描述
private String description;

// 工厂bean(使用场景未研究)
private String factoryBeanName;

// 强制初始化(使用场景未研究)
private boolean enforceInitMethod = true;

// 强制销毁(使用场景未研究)
private boolean enforceDestroyMethod = true;

// 是否是合成bean(使用场景未研究)
private boolean synthetic = false;

// 角色(使用场景未研究)
private int role = BeanDefinition.ROLE_APPLICATION;

// 资源(使用场景未研究)
private Resource resource;
```
可以看到，AbstractBeanDefinition的属性里包含了我们在业务场景分析时的大部分属性，也有很多其他属性，这些属性为spring完成注入和扩展bean所定义的，在后面分析BeanDefinition解析和自动注入时再来看他们的作用。
BeanDefinition有以下子类：

- RootBeanDefinition	所有的Bean定义，在实例化之前都会merge为RootBeanDefinition，在进行处理，提供一个统一的bean定义处理视图
- ChildBeanDefinition	在spring freamwork 4.0的版本中暂时没有看到其怎么用的
- GenericBeanDefinition	我们定义在xml文件和用注解配置的bean在BeanDefinition解析阶段都会解析成GenericBeanDefinition或其子类
