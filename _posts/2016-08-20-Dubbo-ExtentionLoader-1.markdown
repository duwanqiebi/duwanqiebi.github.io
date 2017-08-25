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

### SPI例子

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
        ServiceLoader<Coder> serviceLoader = ServiceLoader.load(Coder.class);
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

### SPI的缺点

> - JDK标准的SPI会一次性实例化扩展点所有实现，如果有扩展实现初始化很耗时，但如果没用上也加载，会很浪费资源。
> - 如果扩展点加载失败，连扩展点的名称都拿不到了。比如：JDK标准的ScriptEngine，通过 getName()获取脚本类型的名称，但如果RubyScriptEngine因为所依赖的jruby.jar不存在，导致RubyScriptEngine类加载失败，这个失败原因被吃掉了，和ruby对应不起来，当用户执行ruby脚本时，会报不支持ruby，而不是真正失败的原因。
> - 增加了对扩展点IoC和AOP的支持，一个扩展点可以直接setter注入其它扩展点。

为了解决这个问题，dubbo实现了自己的一套spi机制，即**ExtensionLoader**


