---
layout:     post
category: [框架学习,dubbo]
title:      "Dubbo源码学习(二)"
subtitle:   "ExtensionLoader的@Adaptive与@Activate"
date:       2017-08-29 11:00:00
author:     "独顽且鄙"
header-img: "img/HelloWorld.jpg"
catalog:    true
tags:
    - dubbo源码
---

## @Adaptive注解

在上一篇ExtensionLoader的博客中记录了,有两种扩展点,一种是普通的扩展实现,另一种就是自适应的扩展点,即@Adaptive注解的实现类。

<pre class="prettyprint">
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE, ElementType.METHOD})
public @interface Adaptive {
    String[] value() default {};
}
</pre>

@Adaptive注解可以加载实现类的类上或方法上。下面可以通过源码去分析,先放结论:

- 注解在实现类上时,提供一种标识表名此实现类并不提供具体的业务实现,而是一个该接口的适配器。例如AdaptiveCompiler、AdaptiveExtensionFactory。
- 注解在接口方法上时,根据某种规则动态生成代码,动态适配。例如Protocol接口

## 主要方法

### T getAdaptiveExtension()

#### 调用流程

<pre class="prettyprint">
T getAdaptiveExtension()         //在cachedAdaptiveInstance缓存中获取,获取不到则进入下一步创建
    -> T createAdaptiveExtension()      
        -> getAdaptiveExtensionClass()      //加载完配置文件后,会在cachedAdaptiveClass缓存中获取,获取不到则创建
            -> getExtensionClasses()          //如果第一次,则加载配置文件
            -> createAdaptiveExtensionClass()   
                -> createAdaptiveExtensionClassCode()   //动态生成代码
                -> Compiler.compile(String code, ClassLoader classLoader)   //编译动态生成的代码
        -> injectExtension(T instance)          //注入
</pre>

cachedAdaptiveClass缓存是@Adaptive注解在类上的缓存,所以上述代码的大致就是:

- 如果@Adaptive注解在类上,则直接返回这个类的实例
- 如果@Adaptive注解在方法上,则动态生成一个类。

#### createAdaptiveExtensionClassCode()

动态生成的代码比较长,但是主要的逻辑就是通过Adaptive注解的value属性去设置该接口的Adaptive的规则，
Dubbo中的服务调用的所有数据均可以从URL获取（Dubbo的URL总线模式），那么需要Dubbo帮我们动态生成adaptive的扩展接口的方法入参必须包含URL，这样才能根据运行状态动态选择具体实现。
一个简单的生成类如下代码:

<pre>
@Adaptive({"key1", "key2"})
String yell(URL url, String s);
</pre>

<pre class="prettyprint">
public String yell(URL arg0, String arg1) {
    if (arg0 == null) {
        throw new IllegalArgumentException("url == null");
    }
    URL url = arg0;
    String extName = url.getParameter("key1", url.getParameter("key2", "impl1"));
    if(extName == null) {
        throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.common.extensionloader.ext1.SimpleExt) name from url(" + url.toString() + ") use keys([key1, key2])");
    }
    SimpleExt extension = (SimpleExt)ExtensionLoader.getExtensionLoader(SimpleExt.class).getExtension(extName);
    return extension.yell(arg0, arg1);
}
</pre>

## @Activate注解

可以看到ExtesionLoader中还有一些与@Active注解相关的方法。

- List<T> getActivateExtension(URL url, String key)
- List<T> getActivateExtension(URL url, String key, String group)
- List<T> getActivateExtension(URL url, String[] values, String group)

可以看出,这些方法都是根据url的某些条件获取一个实现类的列表。@Activate注解主要用处是标注在插件接口实现类上，用来配置该扩展实现类激活条件。
在Dubbo框架里面的Filter的各种实现类都通过Activate标注，用来描述该Filter什么时候生效。下面是一些例子:

- MonitorFilter( **@Activate(group = {Constants.PROVIDER, Constants.CONSUMER})** ) 用来告诉Dubbo框架这个Filter是在服务提供端和消费端会生效的；
- TimeoutFilter( **@Activate(group = Constants.PROVIDER)** )则是只在服务提供端生效，消费端是不会调用该Filter；
- ValidationFilte( **@Activate(group = { Constants.CONSUMER, Constants.PROVIDER }, value = Constants.VALIDATION_KEY, order = 10000)** )
要激活的条件除了在消费端和服务提供端激活，**value** 表明了URL参数中必须包含validation才会被激活。**order** 即激活顺序。

#### getActivateExtension的实现

代码很简单,就做了下简单的注释。

<pre class="prettyprint">
public List&lt;T&gt; getActivateExtension(URL url, String[] values, String group) {
    List&lt;T&gt; exts = new ArrayList&lt;T&gt;();
    List&lt;String&gt; names = values == null ? new ArrayList&lt;String&gt;(0) : Arrays.asList(values);
    if (! names.contains(Constants.REMOVE_VALUE_PREFIX + Constants.DEFAULT_KEY)) {  //"-default"
        getExtensionClasses();      //未加载配置文件的加载一遍
        for (Map.Entry&lt;String, Activate> entry : cachedActivates.entrySet()) {
            String name = entry.getKey();
            Activate activate = entry.getValue();
            if (isMatchGroup(group, activate.group())) {    //如果符合指定的group
                T ext = getExtension(name);
                if (! names.contains(name)
                        && ! names.contains(Constants.REMOVE_VALUE_PREFIX + name) 
                        && isActive(activate, url)) {
                    exts.add(ext);
                }
            }
        }
        Collections.sort(exts, ActivateComparator.COMPARATOR);      //排序
    }
    List&lt;T&gt; usrs = new ArrayList&lt;T&gt;();
    for (int i = 0; i &lt; names.size(); i ++) {
    	String name = names.get(i);
        if (! name.startsWith(Constants.REMOVE_VALUE_PREFIX)
        		&& ! names.contains(Constants.REMOVE_VALUE_PREFIX + name)) {
        	if (Constants.DEFAULT_KEY.equals(name)) {
        		if (usrs.size() &gt; 0) {
         		exts.addAll(0, usrs);
         		usrs.clear();
        		}
        	} else {
         	    T ext = getExtension(name);
         	    usrs.add(ext);
        	}
        }
    }
    if (usrs.size() &gt; 0) {
    	exts.addAll(usrs);
    }
    return exts;
}
</pre>






