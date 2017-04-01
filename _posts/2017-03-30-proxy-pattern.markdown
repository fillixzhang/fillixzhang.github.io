---
layout: post
title: "Proxy Pattern"
date: 2017-03-30
categories: design pattern
---

### Intent
> Provide a placeholder for another object to control access to it.

### Applicable situation

* **Virtual Proxy**: delay the creation of the expensive object until needed.
* **Remote Proxy**: provide a local representation for an object in a different address space.
* **Protection Proxy**: control access to the real object, a filter
* **Smart Proxy**: add additional action when accessing the real object, for example, track the number of references, logging, etc.

### Structure
![](https://sourcemaking.com/files/v2/content/patterns/Proxy1.svg)

The above figure shows the basic structure of proxy pattern:
* **Subject**: Interface representing services provided by the RealObject.
It must implemented by both the RealObject and the Proxy.
* **Proxy**: Placeholder for the RealObject, accomplishes extra work like mentioned in last section.
* **RealObject**: The real object that the proxy represents.
* **Client**: holds a reference to a proxy, so that it can handle the proxy the same way as RealObject.

### Dynamic proxy
The drawback of simple proxy is that if there are many objects need to be proxied, you have to define a corresponding proxy for every object and implement all the needed interfaces.

A **dynamic proxy** is a run-time generated class, implementing one or more interfaces you specified, to create a proxy, use the **Static** method `Proxy.newProxyInstance()` which takes three arguments:
* a class loader
* a list of interfaces you wish the proxy to implement
* an implementation of the interface **InvocationHandler** to intercept hand handle method calls

The proxy will automatically redirect all the method calls to the invocation handler:

![](https://opencredo.com/wp-content/uploads/2015/07/Diagram-proxy-invocation-handler-screenshot-1200x208.png)

You just need to implement the **InvocationHandler** interface and add any additional behavior as you need.
```java
public interface InvocationHandler {
    Object invoke(Object proxy, Method method, Object[] args)
    throws Throwable;
}
```
