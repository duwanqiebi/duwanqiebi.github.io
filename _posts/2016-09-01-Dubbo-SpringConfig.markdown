---
layout:     post
category: [框架学习,dubbo]
title:      "Dubbo源码学习(三)"
subtitle:   "spring配置文件的解析"
date:       2017-09-01 11:00:00
author:     "独顽且鄙"
header-img: "img/HelloWorld.jpg"
catalog:    true
tags:
    - dubbo源码
---

## SpringContainer

dubbo的启动过程即是Dubbo容器(Container)的启动过程。
我们通常在spring配置文件中来对dubbo的各项进行配置,dubbo启动时,SpringContainer会对spring配置文件中的dubbo配置部分进行解析,
生成各种各样的配置bean。

<pre class="prettyprint">
public class SpringContainer implements Container {
    
    public static final String SPRING_CONFIG = "dubbo.spring.config";
    
    public static final String DEFAULT_SPRING_CONFIG = "classpath*:META-INF/spring/*.xml";
    
	public void start() {
        String configPath = ConfigUtils.getProperty(SPRING_CONFIG);
        if (configPath == null || configPath.length() == 0) {
            configPath = DEFAULT_SPRING_CONFIG;
        }
        context = new ClassPathXmlApplicationContext(configPath.split("[,\\s]+"));
        context.start();
    }
}
</pre>

可以看到,SpringContainer启动时,会去找dubbo.spring.config的配置,即spring配置文件的路径,如果没配置,则用默认路径classpath*:META-INF/spring/*.xml,
并启动spring上下文。

## dubbo扩展Spring xml schema

dubbo在spring文件中的配置是利用spring的可扩展xml来实现的,具体用法在spring官方文档中有更为详细的介绍。[https://docs.spring.io/spring/docs/current/spring-framework-reference/html/xml-custom.html](https://docs.spring.io/spring/docs/current/spring-framework-reference/html/xml-custom.html)

#### DubboNamespaceHandler

<pre class="prettyprint">
public class DubboNamespaceHandler extends NamespaceHandlerSupport {
	public void init() {
	    registerBeanDefinitionParser("application", new DubboBeanDefinitionParser(ApplicationConfig.class, true));
        registerBeanDefinitionParser("module", new DubboBeanDefinitionParser(ModuleConfig.class, true));
        registerBeanDefinitionParser("registry", new DubboBeanDefinitionParser(RegistryConfig.class, true));
        registerBeanDefinitionParser("monitor", new DubboBeanDefinitionParser(MonitorConfig.class, true));
        registerBeanDefinitionParser("provider", new DubboBeanDefinitionParser(ProviderConfig.class, true));
        registerBeanDefinitionParser("consumer", new DubboBeanDefinitionParser(ConsumerConfig.class, true));
        registerBeanDefinitionParser("protocol", new DubboBeanDefinitionParser(ProtocolConfig.class, true));
        registerBeanDefinitionParser("service", new DubboBeanDefinitionParser(ServiceBean.class, true));
        registerBeanDefinitionParser("reference", new DubboBeanDefinitionParser(ReferenceBean.class, false));
        registerBeanDefinitionParser("annotation", new DubboBeanDefinitionParser(AnnotationBean.class, true));
    }
}
</pre>

DubboNamespaceHandler类实现了Spring的NamespaceHandlerSupport接口,此接口只有一个方法init(),方法中为spring的命名空间注册解析器。
例如:当spring解析到<dubbo:application>标签时,就会使用new DubboBeanDefinitionParser(ApplicationConfig.class, true)这个对象的parse方法进行解析。

#### DubboBeanDefinitionParser

<pre class="prettyprint">
public class DubboBeanDefinitionParser implements BeanDefinitionParser {
    
    private final boolean required;

    public DubboBeanDefinitionParser(Class&lt;?&gt; beanClass, boolean required) {
        this.beanClass = beanClass;
        this.required = required;
    }
    
    public BeanDefinition parse(Element element, ParserContext parserContext) {
        return parse(element, parserContext, beanClass, required);
    }
}
</pre>

DubboBeanDefinitionParser实现了BeanDefinitionParser接口,其中的parse方法提供了具体的解析方法,并生成bean交由Spring容器进行统一管理

## Dubbo具体解析配置文件流程

在阅读parse方法的过程中,我将解析的流程大体分为了四个部分:

1. 获取beanDefinition ID
2. 注册bean
3. 具体config的不同细节解析
4. 属性注入

#### 获取beanDefinition ID

<pre class="prettyprint">
String id = element.getAttribute("id");
if ((id == null || id.length() == 0) && required) {
	String generatedBeanName = element.getAttribute("name");
	if (generatedBeanName == null || generatedBeanName.length() == 0) {
	    if (ProtocolConfig.class.equals(beanClass)) {
	        generatedBeanName = "dubbo";    //protocol默认id为dubbo协议
	    } else {
	        generatedBeanName = element.getAttribute("interface");  //其他config取默认interface(主要针对serviceBean)
	    }
	}
	if (generatedBeanName == null || generatedBeanName.length() == 0) {
		generatedBeanName = beanClass.getName();    //还为空就取config的全路径类名作为bean的Id
	}
    id = generatedBeanName;
    //如果有多个,id后加递增数字以区分
    int counter = 2;
    while(parserContext.getRegistry().containsBeanDefinition(id)) {
        id = generatedBeanName + (counter ++);
    }
}
</pre>

此部分是为了为BeanDifinition设置ID。具体流程如下:

1. 取配置文件中的id属性,如果为空,往下走
2. 如果beanClass为ProtocolConfig,则id="dubbo";否则取interface属性。如果为空,往下走
3. id = beanClass的全路径名
4. 如果Spring已经有此id的bean了,则在当前id后加递增的数字以区分。

#### 注册bean

<pre class="prettyprint">
if (id != null && id.length() > 0) {
    if (parserContext.getRegistry().containsBeanDefinition(id))  {
		throw new IllegalStateException("Duplicate spring bean id " + id);
	}
    parserContext.getRegistry().registerBeanDefinition(id, beanDefinition); //注册bean
    beanDefinition.getPropertyValues().addPropertyValue("id", id);
}
</pre>

此步骤直接将beanDifinition向spring容器中注册并添加ID属性。

#### 具体config的不同细节解析

<pre class="prettyprint">
if (ProtocolConfig.class.equals(beanClass)) {
    //循环之前加载的bean,如果有ProtocolConfig的属性时,添加
    for (String name : parserContext.getRegistry().getBeanDefinitionNames()) {
        BeanDefinition definition = parserContext.getRegistry().getBeanDefinition(name);
        PropertyValue property = definition.getPropertyValues().getPropertyValue("protocol");
        if (property != null) {
            Object value = property.getValue();
            if (value instanceof ProtocolConfig && id.equals(((ProtocolConfig) value).getName())) {
                definition.getPropertyValues().addPropertyValue("protocol", new RuntimeBeanReference(id));
            }
        }
    }
} else if (ServiceBean.class.equals(beanClass)) {
    String className = element.getAttribute("class");
    if(className != null && className.length() > 0) {
        RootBeanDefinition classDefinition = new RootBeanDefinition();
        classDefinition.setBeanClass(ReflectUtils.forName(className));
        classDefinition.setLazyInit(false);
        parseProperties(element.getChildNodes(), classDefinition);
        beanDefinition.getPropertyValues().addPropertyValue("ref", new BeanDefinitionHolder(classDefinition, id + "Impl"));
    }
} else if (ProviderConfig.class.equals(beanClass)) {
    parseNested(element, parserContext, ServiceBean.class, true, "service", "provider", id, beanDefinition);
} else if (ConsumerConfig.class.equals(beanClass)) {
    parseNested(element, parserContext, ReferenceBean.class, false, "reference", "consumer", id, beanDefinition);
}
</pre>

此部分针对不同的配置项,进行一些不同的细节处理。

1. 如果为ProtocolConfig,则为之前已经注册的且有ProtocolConfig属性的Bean进行赋值
2. 如果为ServiceBean,则以"class"属性(即实现类)的类注册一个BeanDefinition
3. 如果为ProviderConfig或是ConsumerConfig,则进入parseNested解析嵌套的xml配置

#### 属性注入

剩下部分的代码都是为注册的bean设置属性的过程,细节很多,代码也很长。但是逻辑基本都很简单。后续看代码的过程如果有特殊的属性需要记录的,在此记录!




