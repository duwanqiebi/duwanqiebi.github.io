---
layout:     post
category: [框架学习,dubbo]
title:      "Dubbo源码学习(四)"
subtitle:   "注册中心(一)---RegistryFactory"
date:       2017-09-08 11:00:00
author:     "独顽且鄙"
header-img: "img/HelloWorld.jpg"
catalog:    true
tags:
    - dubbo源码
---

## 服务的注册、消费

在使用dubbo的过程中,注册中心起到了不可或缺的作用。服务的治理、发现等都需要依赖于注册中心。Dubbo提供了多种注册中心的实现,例如:Zookeeper、Redis等等。
在提供者注册服务、消费者消费服务时都需要通过注册中心来实现。

**ServiceConfig**

<pre class="prettyprint">
private void doExportUrlsFor1Protocol(ProtocolConfig protocolConfig, List&lt;URL&gt; registryURLs) {
    /* .......省略.......... */
    Exporter&lt;?&gt; exporter = protocol.export(invoker);
    /* .......省略.......... */
}
</pre>


**ReferenceConfig**

<pre class="printprint">
private T createProxy(Map&lt;String, String&gt; map) {
    /* .......省略.......... */
    refprotocol.refer(interfaceClass, url)
    /* .......省略.......... */
}
</pre>

在ServiceConfig、ReferenceConfig两个API中, **protocol** 、 **refprotocol** 的值都是 **ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension()** 。
即Protocol$Adaptive这个动态生成类。在这个动态类中,会取得url.getProtocol(),并获得对应name的Extension。即**RegistryProtocol**。

RegistryProtocol为Protocol接口的实现类,接口的两个重要方法为**export(Invoker<T> invoker)**,**refer(Class<T> type, URL url)**,分别为暴露服务、引用服务。
在RegistryProtocol的这两个方法中,我们先只关注有关注册中心的逻辑。

<pre class="prettyprint">
public class RegistryProtocol implements Protocol {
    private RegistryFactory registryFactory;
    
    public &lt;T&gt; Exporter&lt;T&gt; export(final Invoker&lt;T&gt; originInvoker) throws RpcException {
        ...
        //registry provider
        final Registry registry = getRegistry(originInvoker);
        final URL registedProviderUrl = getRegistedProviderUrl(originInvoker);
        registry.register(registedProviderUrl);     //具体注册中心进行注册url
        ...
    }
    
    private Registry getRegistry(final Invoker&lt;?>&gt;originInvoker){
        URL registryUrl = originInvoker.getUrl();
        if (Constants.REGISTRY_PROTOCOL.equals(registryUrl.getProtocol())) {
            String protocol = registryUrl.getParameter(Constants.REGISTRY_KEY, Constants.DEFAULT_DIRECTORY);
            registryUrl = registryUrl.setProtocol(protocol).removeParameter(Constants.REGISTRY_KEY);
        }
        return registryFactory.getRegistry(registryUrl);
    }
}
</pre>

从RegistryProtocol的代码中可以看出,通过注入的RegistryFactory获取具体Registry对象,通过Registry来完成注册中心的具体的逻辑。

## RegistryFactory

**RegistryFactory**的作用是用来获取Registry对象,RegistryFactory的体系如下图:

[RegistryFactory](http://zqimage.oss-cn-beijing.aliyuncs.com/dubbo%E6%BA%90%E7%A0%81/RegistryFactory.png)

先看下RegistryFactory的代码:

<pre class="prettyprint">
@SPI("dubbo")
public interface RegistryFactory {
    @Adaptive({"protocol"})
    Registry getRegistry(URL url);
}
</pre>

RegistryFactory接口只有一个方法,即获取Registry对象,此方法为@Adaptive方法,会在url里取得**protocol**的值并获取对应的扩展,生成的动态扩展代码应如下:

<pre class="prettyprint">
public class RegistryFactory$Adpative implements RegistryFactory {

    public Registry getRegistry(URL arg0) {
        if (arg0 == null){
            throw new IllegalArgumentException("url == null");
        }

        URL url = arg0;
        String extName = (url.getProtocol() == null ? "dubbo" : url.getProtocol());
        if (extName == null) {
            throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.registry.RegistryFactory) name from url(" + url.toString() + ") use keys([protocol])");
        }

        RegistryFactory extension = ExtensionLoader.getExtensionLoader(RegistryFactory.class).getExtension(extName);
        return extension.getRegistry(arg0);
    }
}
<pre>

#### AbstractRegistryFactory

<pre class="prettyprint>
public Registry getRegistry(URL url) {
	url = url.setPath(RegistryService.class.getName())
			.addParameter(Constants.INTERFACE_KEY, RegistryService.class.getName())
			.removeParameters(Constants.EXPORT_KEY, Constants.REFER_KEY);
	String key = url.toServiceString();
    // 锁定注册中心获取过程，保证注册中心单一实例(锁住create的过程,否则可能重复create)
    LOCK.lock();
    try {
        Registry registry = REGISTRIES.get(key);
        if (registry != null) {
            return registry;
        }
        registry = createRegistry(url);
        if (registry == null) {
            throw new IllegalStateException("Can not create registry " + url);
        }
        REGISTRIES.put(key, registry);
        return registry;
    } finally {
        // 释放锁
        LOCK.unlock();
    }

protected abstract Registry createRegistry(URL url);
</pre>

**AbstractRegistryFactory**是个抽象类,实现了RegistryFactory接口。在**getRegistry**方法中,先去缓存中查找,如果没有,则调用抽象方法**createRegistry(URL url)**进行创建,具体逻辑由其子类来实现。

#### ZookeeperRegistryFactroy

<pre class="prettyprint">
public class ZookeeperRegistryFactory extends AbstractRegistryFactory {
	
	private ZookeeperTransporter zookeeperTransporter;

    public void setZookeeperTransporter(ZookeeperTransporter zookeeperTransporter) {
		this.zookeeperTransporter = zookeeperTransporter;
	}

	public Registry createRegistry(URL url) {
        return new ZookeeperRegistry(url, zookeeperTransporter);
    }

}
</pre>

**ZookeeperRegistryFactroy**直接new了一个ZookeeperRegistry对象。
参数中有个**ZookeeperTransporter**对象,dubbo中实现了两套ZookeeperTransporter实现类,分别基于两种主流的zookeeper客户端,一套是基于zkclient的,一套是基于Curator。

#### MulticastRegistryFactory

<pre class="prettyprint>
public class MulticastRegistryFactory extends AbstractRegistryFactory {

    public Registry createRegistry(URL url) {
        return new MulticastRegistry(url);
    }

}
</pre>

**MulticastRegistryFactory**也是直接new了一个MulticastRegistry对象。

#### DubboRegistryFactory

**TODO**











