---
layout:     post
category: [框架学习,dubbo]
title:      "Dubbo源码学习"
subtitle:   "dubbo的SPI机制-ExtensionLoader"
date:       2017-08-20 11:00:00
author:     "独顽且鄙"
header-img: "img/HelloWorld.jpg"
catalog:    true
tags:
    - dubbo源码
---

## JAVA的SPI机制

#### SPI例子

SPI的全名为Service Provider Interface。先写下例子更好理解：


<pre class="prettyprint">
public interface Coder {
    void code();
}
</pre>

<pre class="prettyprint">
public class JavaCoder implements Coder {
    public void code() {
        System.out.println("code in JAVA");
    }
}
</pre>

<pre class="prettyprint">
public class GoCoder implements Coder {
    public void code() {
        System.out.println("code in GO");
    }
}
</pre>

META-INF/services下创建文件名：com.dubbo.mylearncode.spi.code（即Coder接口的全路径名）

<pre class="prettyprint">
com.dubbo.mylearncode.spi.JavaCoder
com.dubbo.mylearncode.spi.GoCoder
</pre>


<pre class="prettyprint">
public class Main {
    public static void main(String[] args){
        ServiceLoader&lt;Coder&gt; serviceLoader = ServiceLoader.load(Coder.class);
        for(Coder coder : serviceLoader){
            coder.code();
        }
    }
}
</pre>

执行Main类，输出的结果为：


<pre class="prettyprint">
code in JAVA
code in GO
</pre>

通过ServiceLoader.load(Coder.class)方法，java会到META-INF/services目录下寻找Coder接口的全路径名的文件，即*com.dubbo.mylearncode.spi.code*，找到文件后会自动加载配置的实现类，从而能够实现可插拔的接口实现定制。

#### SPI的缺点

> - JDK标准的SPI会一次性实例化扩展点所有实现，如果有扩展实现初始化很耗时，但如果没用上也加载，会很浪费资源。
> - 如果扩展点加载失败，连扩展点的名称都拿不到了。比如：JDK标准的ScriptEngine，通过 getName()获取脚本类型的名称，但如果RubyScriptEngine因为所依赖的jruby.jar不存在，导致RubyScriptEngine类加载失败，这个失败原因被吃掉了，和ruby对应不起来，当用户执行ruby脚本时，会报不支持ruby，而不是真正失败的原因。
> - 增加了对扩展点IoC和AOP的支持，一个扩展点可以直接setter注入其它扩展点。

为了解决这个问题，dubbo实现了自己的一套spi机制，即**ExtensionLoader**

## ExtensionLoader

ExtensionLoader的用法不做即录，直接记录阅读源码的过程。

#### 缓存注释（类属性）


缓存类型 | 缓存名 | 作用
---|---|---
ConcurrentMap<Class<?>, String> | cachedNames | 非@Adaptive的实现类的名字缓存  class -> 扩展点名字(即配置文件中的key)
Holder<Map<String, Class<?>>> | cachedClasses | 非@Adaptive的实现类的名字缓存，与上面相反
Map<String, Activate> |  cachedActivates | @Activate的实现类缓存  名字 -> Activate对象
Class<?> |  cachedAdaptiveClass | @Adaptive的实现类类缓存（从代码中看出,只允许一个Adaptive实现类)
Holder<Object> | cachedAdaptiveInstance | @Adaptive注解的实例缓存
ConcurrentMap<String, Holder<Object>> | cachedInstances | 非@Adaptive实现类的实例缓存
String | cachedDefaultName | 默认扩展的名字，即@SPI的value值


#### 入口


<pre class="prettyprint">
public static &lt;T&gt; ExtensionLoader&lt;T&gt; getExtensionLoader(Class&lt;T&gt; type) {
    if (type == null)
        throw new IllegalArgumentException("Extension type == null");
    if(!type.isInterface()) {
        throw new IllegalArgumentException("Extension type(" + type + ") is not interface!");
    }
    if(!withExtensionAnnotation(type)) {   //如果无SPI注解
        throw new IllegalArgumentException("Extension type(" + type + 
                ") is not extension, because WITHOUT @" + SPI.class.getSimpleName() + " Annotation!");
    }
    
    ExtensionLoader&lt;T&gt; loader = (ExtensionLoader&lt;T&gt;) EXTENSION_LOADERS.get(type);
    if (loader == null) {
        EXTENSION_LOADERS.putIfAbsent(type, new ExtensionLoader&lt;T&gt;(type));
        loader = (ExtensionLoader&lt;T&gt;) EXTENSION_LOADERS.get(type);
    }
    return loader;
}
</pre>

ExtensionLoader的主入口是一个静态方法getExtensionLoader，接受一个class类型的参数，代表着要获取哪个接口的扩展点。ExtensionLoader是通过@SPI注解来标记一个接口是否可以使用ExtensionLoader获取扩展，所以在指定接口没有@SPI注解时会直接抛出异常。EXTENSION_LOADERS是一个static的全局静态缓存，存着class与ExtensionLoader的对应map，所以先会在缓存里找对应的ExtensionLoader，如果找不到，就new一个ExtensionLoader并加入缓存中。下面看看new ExtensionLoader<T>(type)是怎么实现的。


#### 构造方法


<pre class="prettyprint">
private ExtensionLoader(Class type) {
    this.type = type;
    if(type == ExtensionFactory.class){
        objectFactory = null;
    }else{
        objectFactory = ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension();
    }
}
</pre>

进入私有构造方法后，主要对两个属性进行了赋值。type，即ExtensionLoader的泛型类型；objectFactory为ExtensionFactory类型，之后在动态注入时会用到。


