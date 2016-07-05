---
layout:     post
category: [框架学习,dubbo,Rest接口]
title:      "dubbox Rest接口学习笔记"
subtitle:   ""
date:       2016-07-03 11:07:00
author:     "独顽且鄙"
header-img: "img/HelloWorld.jpg"
catalog:    true
tags:
    - dubbo
---

## Rest架构

之前在学习springMVC时候，看到过@RestController这样的一个注解，但是当时也只是知道被@RestController注解的控制器中的方法中返回的内容都会加入到ResponseBody中，而不是返回一个视图，并没有深入的去了解过Rest究竟是什么东西。

#### Rest

REST,是Representational State Transfer的缩写，翻译来就是表现层状态转移。这里有几个非常重要的概念

- **1.资源(Resources):** 资源是Representational State Transfer省略的主语。网络上的一切都可以当作资源，我理解的资源就是数据。
- **2.表现层(Representational):** 表现层就是资源具体的表现形式，通常使用的有JSON，XML，TEXT等。
- **3.转移(Transfer):** 转移就是通过HTTP的动词（GET,POST,PUT,DELETE等），对资源进行具体的操作，使资源的具体状态得到改变，并通过表现层来表现。

#### REST的优点

> 以下摘自维基百科：
>
> - 可更高效利用缓存来提高响应速度
> - 通讯本身的无状态性可以让不同的服务器的处理一系列请求中的不同请求，提高服务器的扩展性
> - 浏览器即可作为客户端，简化软件需求
> - 相对于其他叠加在HTTP协议之上的机制，REST的软件依赖性更小
> - 不需要额外的资源发现机制
> - 在软件技术演进中的长期的兼容性更好
>
> 这里我还想特别补充REST的显著优点：基于简单的文本格式消息和通用的HTTP协议，使它具备极广的适用性，几乎所有语言和平台都对它提供支持，同时其学习和使用的门槛也较低。

知乎上[覃超](https://www.zhihu.com/people/qin.chao)的一个回答把rest的其中一点优势解释的很易懂。[知乎链接](https://www.zhihu.com/question/27785028)

>大家都知道"古代"网页都是前端后端融在一起的，比如之前的PHP，JSP等。在之前的桌面时代问题不大，但是近年来移动互联网的发展，各种类型的Client层出不穷，RESTful可以通过一套统一的接>口为Web，iOS和Android提供服务。另外对于广大平台来说，比如Facebook platform，微博开放平台，微信公共平台等，
>它们不需要有显式的前端，只需要一套提供服务的接口，于是RESTful更是它们最好的选择。在RESTful架构下：
>![](https://pic2.zhimg.com/06ee404783540f0af299042057738a99_b.jpg)

## Dubbo与Dubbox

#### Dubbo简介

[Dubbo](dubbo.io)是阿里开源的一款分布式服务框架，致力于提供高性能和透明化的RPC远程服务调用方案。上周在新公司才算第一次正式的接触dubbo、分布式、SOA等等的概念。这些将来还有待学习和理解，但是现在有更亟待学习的东西－－使用duboox开发Rest接口

#### Dubbox简介

[Dubbox](http://dangdangdotcom.github.io/dubbox/)是当当网基于dubbo的扩展。扩展内容包括但不限于：

>- **支持REST风格远程调用（HTTP + JSON/XML)：** 基于非常成熟的[JBoss RestEasy](http://resteasy.jboss.org/)框架，在dubbo中实现了REST风格（HTTP + JSON/
>XML）的远程调用，以显著简化企业内部的跨语言交互，同时显著简化企业对外的Open 
>API、无线API甚至AJAX服务端等等的开发。事实上，这个REST调用也使得Dubbo可以对当今特别流行的“微服务”架构提供基础性支持。 
>另外，REST调用也达到了比较高的性能，在基准测试下，HTTP + JSON与Dubbo 2.x默认的RPC协议（即TCP + Hessian2二进制序列化）之间只有1.5倍左右的差距，详见文档中的基准测试报告。
>
>- **支持基于Kryo和FST的Java高效序列化实现** 基于当今比较知名的[Kryo](https://github.com/EsotericSoftware/kryo)和[FST](https://github.com/RuedigerMoeller/
>fast-serialization)高性能序列化库，为Dubbo默认的RPC协议添加新的序列化实现，并优化调整了其序列化体系，比较显著的提高了Dubbo RPC的性能，详见文档中的基准测试报告。
>
>- **支持基于Jackson的JSON序列化：** 基于业界应用最广泛的[Jackson](http://www.codehaus.org/)序列化库，为Dubbo默认的RPC协议添加新的JSON序列化实现。
>
>- **支持基于嵌入式Tomcat的HTTP remoting体系：** 基于嵌入式tomcat实现dubbo的HTTP 
>remoting体系（即dubbo-remoting-http），用以逐步取代Dubbo中旧版本的嵌入式Jetty，可以显著的提高REST等的远程调用性能，并将Servlet 
>API的支持从2.5升级到3.1。（注：除了REST，dubbo中的WebServices、Hessian、HTTP Invoker等协议都基于这个HTTP remoting体系）。
>
>- **升级Spring：** 将dubbo中Spring由2.x升级到目前最常用的3.x版本，减少版本冲突带来的麻烦。
>
>- **升级ZooKeeper客户端：** 将dubbo中的zookeeper客户端升级到最新的版本，以修正老版本中包含的bug。
>
>- **支持完全基于Java代码的Dubbo配置：** 基于Spring的Java Config，实现完全无XML的纯Java代码方式来配置dubbo
>
>- **调整Demo应用：**  暂时将dubbo的demo应用调整并改写以主要演示REST功能、Dubbo协议的新序列化方式、基于Java代码的Spring配置等等。
>
>- **修正了dubbo的bug **  包括配置、序列化、管理界面等等的bug。
>
>**注：dubbox和dubbo 2.x是兼容的，没有改变dubbo的任何已有的功能和配置方式（除了升级了spring之类的版本）**


未完待续

