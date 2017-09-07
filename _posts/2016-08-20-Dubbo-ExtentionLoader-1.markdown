---
layout:     post
category: [框架学习,dubbo]
title:      "Dubbo源码学习(一)"
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
--- | --- | ---
ConcurrentMap&lt;Class&lt;?&gt;, String&gt; | cachedNames | 非@Adaptive的实现类的名字缓存  class -> 扩展点名字(即配置文件中的key)
Holder&lt;Map&lt;String, Class&lt;?&gt;&gt;&gt; | cachedClasses | 非@Adaptive的实现类的名字缓存，与上面相反
Map&lt;String, Activate&gt; |  cachedActivates | @Activate的实现类缓存  名字 -> Activate对象
Class&lt;?&gt; |  cachedAdaptiveClass | @Adaptive的实现类类缓存（从代码中看出,只允许一个Adaptive实现类)
Holder&lt;Object&gt; | cachedAdaptiveInstance | @Adaptive注解的实例缓存
ConcurrentMap&lt;String, Holder&lt;Object&gt;&gt; | cachedInstances | 非@Adaptive实现类的实例缓存
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

ExtensionLoader的主入口是一个静态方法getExtensionLoader，接受一个class类型的参数，代表着要获取哪个接口的扩展点。ExtensionLoader是通过@SPI注解来标记一个接口是否可以使用ExtensionLoader获取扩展，所以在指定接口没有@SPI注解时会直接抛出异常。

EXTENSION_LOADERS是一个static的全局静态缓存，存着class与ExtensionLoader的对应map，所以先会在缓存里找对应的ExtensionLoader，如果找不到，就new一个ExtensionLoader并加入缓存中。下面看看new ExtensionLoader<T>(type)是怎么实现的。


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


## 主要方法


<pre class="prettyprint">
private static final ExtensionLoader&lt;Container&gt; loader = ExtensionLoader.getExtensionLoader(Container.class);
</pre>

上面一行代码是在com.alibaba.dubbo.container.Main获取ExtensionLoader的代码，在这行代码执行之后，loader中的关于扩展点的缓存还尚未加载，那缓存是什么时候加载的呢？ExtensionLoader针对spi机制进行了扩展，只有在真正用到的时候才会进行加载。

### T getExtension(String name) 根据name获取类实例对象


<pre class="prettyprint">
getExtension(name)    //在cachedInstances缓存中找，找不到进入下面
    -> createExtension(name)
        -> getExtensionClasses()  //在cachedClasses缓存中找，找不到进入下面
            -> loadExtensionClasses()
                ->loadFile(extensionClasses,filePath)  //读取文件，填充缓存
    -> injectExtension(T instance)
    
</pre>

#### loadFile()

loadFile是ExtensionLoader加载spi配置文件的方法，是一个比较重要的方法，不只是在getExtension(String name)方法中用到，在所有类似的方法（例如：getExtensionClass(String name),getDefaultExtensionName()等）缓存还未初始化时都会调用。

加载配置文件时，dubbo会从三个位置去找文件，即：
- META-INF/services/
- META-INF/dubbo/
- META-INF/services/internal/
但是dubbo默认都将配置文件放到了META-INF/services/internal/路径下面


下面是loadFile方法中解析class的其中较关键的部分:

<pre class="prettyprint">
if (clazz.isAnnotationPresent(Adaptive.class)) {    //如果该实现类有@Adaptive注解
    if(cachedAdaptiveClass == null) {
        cachedAdaptiveClass = clazz;
    } else if (! cachedAdaptiveClass.equals(clazz)) {
        throw new IllegalStateException("More than 1 adaptive class found: "
            + cachedAdaptiveClass.getClass().getName()
            + ", " + clazz.getClass().getName());
    }
} else {    //如果该实现类无@Adaptive注解
    try {
        clazz.getConstructor(type); //查看该实现类是否有type的构造方法,没有则直接进入异常
        //如果有type类型的构造方法,则为wrapper类。加入wrapper缓存
        Set&lt;Class&lt;?&gt;&gt; wrappers = cachedWrapperClasses;
        if (wrappers == null) {
            cachedWrapperClasses = new ConcurrentHashSet&lt;Class&lt;?&gt;&gt;();
            wrappers = cachedWrapperClasses;
        }
        wrappers.add(clazz);
} catch (NoSuchMethodException e) {     //没有type类型的构造方法
    clazz.getConstructor();
    if (name == null || name.length() == 0) {
        name = findAnnotationName(clazz);
        if (name == null || name.length() == 0) {
            if (clazz.getSimpleName().length() &gt; type.getSimpleName().length()
                && clazz.getSimpleName().endsWith(type.getSimpleName())) {
                name = clazz.getSimpleName().substring(0, clazz.getSimpleName().length() - type.getSimpleName().length()).toLowerCase();
            } else {
                throw new IllegalStateException("No such extension name for the class " + clazz.getName() + " in the config " + url);
            }
        }
    }
    String[] names = NAME_SEPARATOR.split(name);
    if (names != null && names.length > 0) {
        Activate activate = clazz.getAnnotation(Activate.class);
        if (activate != null) {
            cachedActivates.put(names[0], activate);
        }
        for (String n : names) {
            if (! cachedNames.containsKey(clazz)) {
                cachedNames.put(clazz, n);
            }
            Class&lt;?&gt; c = extensionClasses.get(n);
            if (c == null) {
                extensionClasses.put(n, clazz);
            } else if (c != clazz) {
                throw new IllegalStateException("Duplicate extension " + type.getName() + " name " + n + " on " + c.getName() + " and " + clazz.getName());
            }
        }
    }
}
</pre>

可以看出，实现类被分成两种类型进行了解析。有@Adaptive注解、无@Adaptive注解。而无@Adaptive注解又分为两种，Wrapper与普通实现类，wrapper即对普通的实现类又进行了一层包装，必要特征即：构造方法有一个当前类的参数。
1. 如果实现类有@Adaptive注解，则直接将cachedAdaptiveClass赋值。（如果之前有值且与当前值不相等，直接抛异常）
2. 如果为wrapper类，则加入cachedWrapperClasses缓存
3. 其余的即为普通实现类，则加入cachedNames缓存。另，如果有@Active注解的，则加入cachedActivates缓存。（@Active注解还不太明白，后续看代码看到的时候继续补充）
4. 最后loadFile方法结束后，会将name-class的缓存加入cachedClasses（getExtensionClasses()方法中）

另:如果是有多个Wrapper实现类时,是会进行多层包装的。例如Protocol接口,可能最外层是 **ProtocolFilterWrapper** 包装了 **ProtocolListenerWrapper** ,而 **ProtocolListenWrapper** 里面才包装着具体的实现类(例如 **DubboProtocol** )。

#### createExtension(String name)  EXtensionLoader的实例化

之前说到SPI的其中一个缺点就是一次会实例化所有的配置类。而在createExtension(String name)方法中可以看出，ExtensionLoader在使用一个类的时候才会去实例化这个类。


<pre class="prettyprint">
T instance = (T) EXTENSION_INSTANCES.get(clazz);
if (instance == null) {
    EXTENSION_INSTANCES.putIfAbsent(clazz, (T) clazz.newInstance());
    instance = (T) EXTENSION_INSTANCES.get(clazz);
}
injectExtension(instance);  //注入实例属性
Set&lt;Class&lt;?&gt;&gt; wrapperClasses = cachedWrapperClasses;
//如果有wrapper,为wrapper,否则直接返回。后面的wrapper会覆盖前面的wrapper
if (wrapperClasses != null && wrapperClasses.size() &gt; 0) {
    for (Class&lt;?&gt; wrapperClass : wrapperClasses) {
        instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));
    }
}
return instance;
</pre>


方法首先会到EXTENSION_INSTANCES缓存中取得对象实例，如果没有，则直接利用反射创建一个新对象，并进行属性的注入。如果有wrapper，则创建wrapper对象。通过Wrapper类可以把所有扩展点公共逻辑移至Wrapper中。新加的Wrapper在所有的扩展点上添加了逻辑，有些类似AOP（Wraper代理了扩展点） （目前对wrapper理解还不是特别深刻，后续看代码继续补充）




#### injectExtension(T instance) ExtensionLoader的自动注入


<pre class="prettyprint">
for (Method method : instance.getClass().getMethods()) {
    if (method.getName().startsWith("set")
         && method.getParameterTypes().length == 1
          && Modifier.isPublic(method.getModifiers())) {  //寻找setter方法
        Class&lt;?&gt; pt = method.getParameterTypes()[0];
        try {
            String property = method.getName().length() &gt; 3 ? method.getName().substring(3, 4).toLowerCase() + method.getName().substring(4) : "";
            Object object = objectFactory.getExtension(pt, property);
            if (object != null) {
                method.invoke(instance, object);
            }
        } catch (Exception e) {
            logger.error("fail to inject via method " + method.getName()
                   + " of interface " + type.getName() + ": " + e.getMessage(), e);
        }
    }
}
</pre>

ExtensionLoader会寻找符合条件的setter方法，并利用objectFactory获取对象进行注入。这里获取的对象是通过getAdaptiveExtension()方法获取的自适应对象。@Adaptive我会单独记录一篇文章。

## 其他常用方法（非Adaptive、Active）

- T getDefaultExtension() 获取默认的扩展。
- String getDefaultExtensionName() 获取默认扩展名，即SPI注解中配置的值
- boolean hasExtension(String name)  是否有指定的扩展
- Set<String> getSupportedExtensions() 获取所有支持的扩展


