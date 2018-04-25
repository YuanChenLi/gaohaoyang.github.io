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




## RPC简介 ##
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

## 代码实现 ##
定义接口HelloService：

```java
package org.yunxuan.dubbo.study.common.api;

public interface HelloService {
   	String sayHello(String name);
}
```

定义接口HelloService实现：
```java
package org.yunxuan.dubbo.study.common.impl;

import org.yunxuan.dubbo.study.common.api.HelloService;

public class HelloServiceImpl implements HelloService {
    public String sayHello(String name) {
        return "Hello " + name;
    }
}
```

RPC框架代码：
```java
package org.yunxuan.dubbo.study.rpc.framework;

import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;
import java.net.InetSocketAddress;
import java.net.ServerSocket;
import java.net.Socket;

public class RpcFramework {

    /**
     * 暴露服务
     *
     * @param service 服务实现
     * @param port    服务端口
     * @throws Exception
     */
    public static void export(final Object service, int port) throws Exception {
        if (service == null)
            throw new IllegalArgumentException("service instance == null");
        if (port <= 0 || port > 65535)
            throw new IllegalArgumentException("Invalid port " + port);
        ServerSocket server = new ServerSocket();
        server.bind(new InetSocketAddress(port));
        System.out.println("Export service " + service.getClass().getName() + " on port " + port);
        while (true) {
            Socket socket = server.accept();
            try {
                try {
                    ObjectInputStream input = new ObjectInputStream(socket.getInputStream());
                    try {
                        String methodName = input.readUTF();
                        Class<?>[] parameterTypes = (Class<?>[]) input.readObject();
                        Object[] arguments = (Object[]) input.readObject();

                        ObjectOutputStream output = new ObjectOutputStream(socket.getOutputStream());
                        try {
                            Method method = service.getClass().getMethod(methodName, parameterTypes);
                            Object result = method.invoke(service, arguments);
                            output.writeObject(result);
                        } catch (Throwable t) {
                            output.writeObject(t);
                        } finally {
                            output.close();
                        }
                    } finally {
                        input.close();
                    }
                } finally {
                    socket.close();
                }
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }

    /**
     * 引用服务
     *
     * @param <T>            接口泛型
     * @param interfaceClass 接口类型
     * @param host           服务器主机名
     * @param port           服务器端口
     * @return 远程服务
     * @throws Exception
     */
    @SuppressWarnings("unchecked")
    public static <T> T refer(final Class<T> interfaceClass, final String host, final int port) throws Exception {
        if (interfaceClass == null)
            throw new IllegalArgumentException("Interface class == null");
        if (!interfaceClass.isInterface())
            throw new IllegalArgumentException("The " + interfaceClass.getName() + " must be interface class!");
        if (host == null || host.length() == 0)
            throw new IllegalArgumentException("Host == null!");
        if (port <= 0 || port > 65535)
            throw new IllegalArgumentException("Invalid port " + port);

        System.out.println("Get remote service " + interfaceClass.getName() + " from server " + host + ":" + port);
        return (T) Proxy.newProxyInstance(interfaceClass.getClassLoader(), new Class<?>[]{interfaceClass}, new InvocationHandler() {
            public Object invoke(Object proxy, Method method, Object[] arguments) throws Throwable {
                Socket socket = new Socket(host, port);
                try {
                    ObjectOutputStream output = new ObjectOutputStream(socket.getOutputStream());
                    try {
                        output.writeUTF(method.getName());
                        output.writeObject(method.getParameterTypes());
                        output.writeObject(arguments);

                        ObjectInputStream input = new ObjectInputStream(socket.getInputStream());
                        try {
                            Object result = input.readObject();
                            if (result instanceof Throwable) {
                                throw (Throwable) result;
                            }
                            return result;
                        } finally {
                            input.close();
                        }
                    } finally {
                        output.close();
                    }
                } finally {
                    socket.close();
                }
            }
        });
    }
}
```

服务端：
```java
package org.yunxuan.dubbo.study.rpc.framework;

import org.yunxuan.dubbo.study.common.api.HelloService;
import org.yunxuan.dubbo.study.common.impl.HelloServiceImpl;

public class RpcProvider {
    public static void main(String[] args) throws Exception {
        HelloService service = new HelloServiceImpl();
        RpcFramework.export(service, 1234);
    }
} 
```

客户端：
```java
package org.yunxuan.dubbo.study.rpc.framework;

import org.yunxuan.dubbo.study.common.api.HelloService;

public class RpcConsumer {

    public static void main(String[] args) throws Exception {
        HelloService service = RpcFramework.refer(HelloService.class, "127.0.0.1", 1234);
        String hello = service.sayHello("World");
        System.out.println(hello);
    }
}
```

先执行服务端，运行结果：
```
Export service org.yunxuan.dubbo.study.common.impl.HelloServiceImpl on port 1234
```

再执行客户端，运行结果：
```
Get remote service org.yunxuan.dubbo.study.common.api.HelloService from server 127.0.0.1:1234
Hello World
```
