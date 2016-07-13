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

在rest服务中,我们通常会接收一个xml或json格式的参数报文,并将返回以xml或json格式返回给调用者。

REST的底层实现会在service的对象和JSON/XML数据格式之间自动做序列化/反序列化。例如,一个简单的User类:

<pre>
public class User implements Serializable {
    
    private String id;
    private String name;
      
    //省略构造方法和getter/setter
}
</pre>

Producer中的接口如下:
<pre>
@Post
@Path("/getuser")
User getUser(User user){
    return user;
}
</pre>

## 校验输入参数



#### 使用Java标准（JSR 303)校验输入参数

TODO

#### 自定义验证错误返回信息

TODO

## 自定义Exception处理

TODO

## 获取上下文(Context)信息

TODO

#### 应用（filter校验Header信息）

TODO
