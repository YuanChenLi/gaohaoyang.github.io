---
layout: post
title:  "Dubbo学习笔记（一）：从一个RPC框架开始"
categories: Dubbo
tags:  dubbo soa rpc
author: 云轩
---

* content
{:toc}

已经使用Dubbo两年多，积累了使用和配置调优方面的经验，但一直没深入了解过Dubbo的原理。由于偶然看到梁飞大神的一篇博客，提起了学习Dubbo框架的兴趣，在学习过程中收获了很多，在此记录。由于本人才疏学浅，不对的地方还望斧正。




# RPC简介 #
>RPC，全称为Remote Procedure Call，即远程过程调用，它是一个计算机通信协议。它允许像调用本地服务一样调用远程服务。它可以有不同的实现方式。如RMI(远程方法调用)、Hessian、Http invoker等。另外，RPC是与语言无关的。

RPC调用流程图![RPC示意图](https://i.imgur.com/GGRqTuN.png)

1. 服务消费方（client）调用以本地调用方式调用服务；
2. client stub接收到调用后负责将方法、参数等组装成能够进行网络传输的消息体；
3. client stub找到服务地址，并将消息发送到服务端；
4. server stub收到消息后进行解码；
5. server stub根据解码结果调用本地的服务；
6. 本地服务执行并将结果返回给server stub；
7. server stub将返回结果打包成消息并发送至消费方；
8. client stub接收到消息，并进行解码；
9. 服务消费方得到最终结果。

# 代码实现 #
定义接口：

```java
package org.yunxuan.dubbo.study.common.api;

public interface HelloService {
   	String sayHello(String name);
}
```
定义接口实现：
```java
package org.yunxuan.dubbo.study.common.impl;

import org.yunxuan.dubbo.study.common.api.HelloService;

public class HelloServiceImpl implements HelloService {
    public String sayHello(String name) {
        return "Hello " + name;
    }
}
```
