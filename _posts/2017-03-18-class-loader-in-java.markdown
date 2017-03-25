---
layout: post
title: "Class loader in Java"
date: 2017-03-18
categories: java
---

## What is class loader
Before a class can be used, it needs to be loaded into the JVM. Class loader is the subsystem which is used to load ‘.class’ files and then create  *java.lang.Class* object from the byte codes in java at runtime.
In practice, '.class' file could be from file system compiled with *javac*, network or any other sources.

## Default class loader
When JVM starts, it will create three default class loaders, each has a parent (except the root class loader) and a predefined location to load class files from:

* `Bootstrap` class loader : load standard jdk class files from jre/lib/rt.jar. It is parent of all class loaders in java and doesn’t have any parents itself.
* `Extension` class loader: load class from jre/lib/ext or other directories pointed by java.ext.dirs system property
* `Application` class loader : load application specific classes from $CLASSPATH,  -classpath command line option or Class-Path attribute of Manifest in JAR.

Here's an example:


```java
public class ClassLoaderTest {

    public static void main(String[] args) {
        System.out.println("Class loader for HashMap: " +
            java.util.HashMap.class.getClassLoader());

        System.out.println("Class loader for DNSNameService: " +
            sun.net.spi.nameservice.dns.DNSNameService.class.getClassLoader());

        System.out.println("Class loader for this class: " +
            ClassLoaderTest.class.getClassLoader());
    }
}
```
```
Class loader for HashMap: null
Class loader for DNSNameService: sun.misc.Launcher$ExtClassLoader@266474c2
Class loader for this class: sun.misc.Launcher$AppClassLoader@4b67cf4d
```
All java class loaders are implemented using *java.lang.Classloader* class except the bootstrap class loader.

## How class loader works
### Delegation model
Each class loader has a reference to its parent class loader, when a class
loader is requested to load a class, the task is recursively delegated to its parent class loader before attempting to find the class itself. If the parent class loader fails to find the requested class, the current class loader will load and find the class itself. It that fails again, a **ClassNotFoundException** will be thrown. The following picture demonstrates the logic:

![](http://blog.cask.co/wp-content/uploads/sites/2/image01.png)

### The java.lang.ClassLoader class

*java.lang.ClassLoader* is an abstract class which provides a series of methods to fulfill the class loading related requirements. Here lists some of the fundamental ones:
should be the parent class if you want to define your own class loader.

|Method|Description|
|------|------------|
|loadClass(String name)|Load class with the specified binary name|
|findClass(String name)|Invoked by loadClass() after checking the parent class loader. Should be overridden by custom class loader implementation
|defineClass(String name, byte[] b, int off, int len)|Converts an array of bytes into an instance of class **Class**|
|resolve(Class<?> c)|Links the specified class|

Of the above mentioned methods, `loadClass(Sting)` is in charge of the whole process, the default implementation of this method searches for classes in the following order:

* Invoke `findLoadedClass(String)` to check if the class has already been loaded.
* Invoke the `loadClass(String)` method on the parent class loader.  If the parent is null the class loader built-in to the virtual machine is used, instead.
* Invoke the `findClass(String)` method to find the class and further call `defineClass()` method which coverts the bytes into a **Class** object.
* If the class was found using the above steps, invoke the `resolveClass(Class)` method on the resulting **Class** object to resolve all the referenced classes if the resolve flag is true.

With this process, a class that has already been loaded by parent class loader should not be loaded by the child again, so child class loader can 'see' the classes loaded by parent class loader, but vice versa is not true.

## Write your own class loader
### Motivation
[See this answer from stackoverflow](http://stackoverflow.com/a/10829369/4318334)

### Custom ClassLoader
```java
public class MyClassLoader extends ClassLoader{

    private String rootDir;

    public MyClassLoader(ClassLoader parent) {
        super(parent);
    }

    public String getRootDir() {
        if (rootDir == null)
            rootDir = "";
        return rootDir;
    }

    public void setRootDir(String rootDir) {
        this.rootDir = rootDir;
    }

    @Override
    public Class<?> loadClass(String name) throws ClassNotFoundException {
        System.out.println("Start to load class " + name);
        return super.loadClass(name);
    }

    @Override
    public Class<?> findClass(String name) {
        String classFilePath = nameToPath(name);
        byte[] classData = readClassBytes(classFilePath);
        Class<?> clazz = defineClass(name, classData, 0, classData.length);
        resolveClass(clazz);
        return clazz;
    }

    private String nameToPath(String name) {
        return getRootDir() + File.separator +
                name.replace('.', File.separatorChar) + ".class";
    }

    private byte[] readClassBytes(String path) {
        File file = new File(path);
        if (!file.exists()) {
            System.out.println("File not exist");
            return null;
        }

        FileChannel channel = null;
        FileInputStream input = null;
        try {
            input = new FileInputStream(file);
            channel = input.getChannel();
            ByteBuffer byteBuffer = ByteBuffer.allocate((int)channel.size());
            while (channel.read(byteBuffer) > 0)
                ;
            return byteBuffer.array();
        } catch (FileNotFoundException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            try {
                if (channel != null)
                    channel.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
            try {
                if (input != null)
                    input.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
        return null;
    }

    public static void main(String[] args) {
        MyClassLoader classLoader1 =
                new MyClassLoader(MyClassLoader.getSystemClassLoader());
        MyClassLoader classLoader2 = new MyClassLoader(null);
        classLoader2.setRootDir("/Users/fillix/code/myproject/jvm/target/classes");

        try {
            Class class1 = classLoader1.loadClass(Foo.class.getName());
            Class class2 = classLoader2.loadClass(Foo.class.getName());
            System.out.println(class1 == class2);
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
    }
}

class Foo {}
```
Notice that only when both the class name and **defining** class loader of the class are same, JVM treats the two **Class** as same. So in the code above, class1 and class2 are actually loaded by different class loaders and they are different objects.

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
