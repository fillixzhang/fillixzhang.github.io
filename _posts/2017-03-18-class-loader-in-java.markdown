---
layout: post
title: "Class loader in Java"
date: 2017-03-18
categories: java
---

## What is class loader
Before a class can be used, it needs to be loaded into the JVM. Class loader is the subsystem which is used to load ‘.class’ files from file system, network or any other sources and then create  *java.lang.Class* object from the byte codes in java at runtime.

## Default class loader
When JVM starts, it will create three default class loaders, each has parent (except the root class loader) and a predefined location to load class files from:

* `Bootstrap` class loader : load standard jdk class files from jre/lib/rt.jar. It is parent of all class loaders in java and doesn’t have any parents itself.
* `Extension` class loader: load class from jre/lib/ext or other directories pointed by java.ext.dirs system property
* `Application` class loader : load application specific classes from $CLASSPATH,  -classpath command line option or Class-Path attribute of Manifest in JAR.

All java class loaders are implemented using *java.lang.Classloader* except the bootstrap class loader.

## How class loader works
### Delegation model
Each class loader has a reference to its parent class loader, when a class
loader is requested to load a class, the task is recursively delegated to its parent class loader before attempting to find the class itself. If the parent class loader fails to find the requested class, the current class loader will find and load the class itself. It that fails again, a **ClassNotFoundException** will be thrown. The following picture demonstrates the logic:

![](http://blog.cask.co/wp-content/uploads/sites/2/image01.png)

Child class loader can see the classes loaded by parent class loader but vice versa is not true.  Also a class loaded by parent should not be loaded by child class loader again.

### The java.lang.ClassLoader class
The *java.lang.ClassLoader* is an abstract class which should be the parent class if you want to define your own class loader. Let's take a look at the default `loadClass()` implementation.
``` java
protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
{
    synchronized (getClassLoadingLock(name)) {
        // First, check if the class has already been loaded
        Class<?> c = findLoadedClass(name);
        if (c == null) {
            long t0 = System.nanoTime();
            try {
                if (parent != null) {
                    c = parent.loadClass(name, false);
                } else {
                    c = findBootstrapClassOrNull(name);
                }
            } catch (ClassNotFoundException e) {
                // ClassNotFoundException thrown if class not found
                // from the non-null parent class loader
            }

            if (c == null) {
                // If still not found, then invoke findClass in order
                // to find the class.
                long t1 = System.nanoTime();
                c = findClass(name);

                // this is the defining class loader; record the stats
                sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                sun.misc.PerfCounter.getFindClasses().increment();
            }
        }
        if (resolve) {
            resolveClass(c);
        }
        return c;
    }
}
```
First it will check if the class requested has already been loaded, if not it will delegate the task to its parent. Given that fails it will call `findClass()` method which searches the class by itself and further calls `defineClass()` method which coverts the bytes into a **Class** object. After that, it will resolve all the referenced classes if needed.

## Related Errors and Exception
### ClassNotFoundException
>Thrown when an application tries to load in a class through its string name using:
>* The `forName` method in class Class.
>* The `findSystemClass` method in class ClassLoader.
>* The `loadClass` method in class ClassLoader.
>
>but no definition for the class with the specified name could be found

### NoClassDefFoundError
>Thrown if the Java Virtual Machine or a ClassLoader instance tries to load in the definition of a class (as part of a normal method call or as part of creating a new instance using the new expression) and no definition of the class could be found.
>
>The searched-for class definition existed when the currently executing class was compiled, but the definition can no longer be found.

When the source was successfully compiled but at runtime the required .class file could not be found, the *NoClassDefFoundError* occurs.

### ClassFormatError
>Thrown when the Java Virtual Machine attempts to read a class file and determines that the file is malformed or otherwise cannot be interpreted as a class file.
