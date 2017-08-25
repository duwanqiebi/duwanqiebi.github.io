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

# JAVA的SPI机制

## SPI例子

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


