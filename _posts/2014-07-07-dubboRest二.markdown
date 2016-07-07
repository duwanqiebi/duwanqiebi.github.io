---
layout:     post
category: [框架学习,dubbo,Rest接口]
title:      "dubbox Rest接口学习笔记(二)"
subtitle:   "实际开发中的一些关键问题"
date:       2016-07-07 11:00:00
author:     "独顽且鄙"
header-img: "img/HelloWorld.jpg"
catalog:    true
tags:
    - dubbo
---

## 定制序列化

TODO

## 校验输入参数



#### 使用Java标准的bean validation annotation（JSR 303)校验输入参数

TODO

#### 自定义验证错误返回信息

TODO

## 自定义Exception处理

TODO

## 获取上下文(Context)信息

TODO

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
>- **修正了dubbo的bug**  包括配置、序列化、管理界面等等的bug。
>
>**注：dubbox和dubbo 2.x是兼容的，没有改变dubbo的任何已有的功能和配置方式（除了升级了spring之类的版本）**


## 搭建开发环境

我利用springBoot和dubbox搭建了一个开发环境。下一篇博客将记录我在实际工作中遇到的一些问题。
