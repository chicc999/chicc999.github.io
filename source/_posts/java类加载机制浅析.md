---
title: java类加载机制浅析
date: 2017-03-20 17:19:04
tags:
- jvm
- java
categories: 编程
comments: false
---

*本文简要归纳java的类加载过程，介绍类加载器，以及怎样破坏类加载器的双亲委派模型*
<!--more-->

# 类加载时机

# 类加载过程

# 类加载器

类加载阶段需要“通过一个类的全限定名来获取描述此类的二进制字节流”，这个动作的实现代码就是类加载器。这个动作放在Java虚拟机外部实现，以便让程序自己决定如何获取所需要的类。

## 类与类加载器

* 类在Java虚拟机中的唯一性由类加载器和这个类本身一同确立。
* 比较两个类（equals，isAssignableFrom,isInstance），要在这两个类是同一个类加载器加载的情况下才有意义。

## 类加载器类型

### 启动类加载器(BootstrapClassLoader)

* 负责加载存放在 <JAVA_HOME>\lib下，或被 -Xbootclasspath参数指定的路径中的，并且能被虚拟机识别的类库（如rt.jar）。
* 启动类加载器是无法被Java程序直接引用。

### 扩展类加载器（ExtensionClassLoader）

* 由 sun.misc.Launcher$ExtClassLoader实现，它负责加载 <JAVA_HOME>\lib\ext目录中，或者由 java.ext.dirs系统变量指定的路径中的所有类库（如javax.开头的类）
* 开发者可以直接使用扩展类加载器。

### 应用程序类加载器(ApplicationClassLoader)

* 由 sun.misc.Launcher$AppClassLoader来实现，它负责加载用户类路径（ClassPath）所指定的类，开发者可以直接使用该类加载器
* 如果应用程序中没有自定义过自己的类加载器，一般情况下这个就是程序中默认的类加载器
* ClassLoader中getSystemClassLoader()的返回值

### 自定义类加载器

如有必要，可以自定义类加载器。例如从网络上接收一段字节流，以实现远程执行代码的功能。

## 双亲委派模型

![双亲委派](http://ovor60v7j.bkt.clouddn.com/blog/classLoader/%E5%8F%8C%E4%BA%B2%E5%A7%94%E6%B4%BE.png)

双亲委派模型如图所示，除了顶层的启动类加载器以外，其余的所有类加载器都应当有自己的父类加载器。

### 工作流程

```java
    protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
    {
        synchronized (getClassLoadingLock(name)) {
            // 判断类是否已经被加载
            Class<?> c = findLoadedClass(name);
            if (c == null) {
                long t0 = System.nanoTime();
                try {
                    if (parent != null) {
                    //如果存在父类加载器，就委派给父类加载器加载
                        c = parent.loadClass(name, false);
                    } else {
                    //不存在父类加载器，交给启动类加载器
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
                    // ClassNotFoundException thrown if class not found
                    // from the non-null parent class loader
                }

                if (c == null) {
                    long t1 = System.nanoTime();
                    // 如果父类加载器和启动类加载器都不能完成加载任务，调用自身的加载功能
                    c = findClass(name);

                    // this is the defining class loader; record the stats
                    sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                    sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                    sun.misc.PerfCounter.getFindClasses().increment();
                }
            }
            if (resolve) {
            //如果需要装入此类引用的其它类
                resolveClass(c);
            }
            return c;
        }
    }
```

类加载过程见代码中注释，由此可以看出流程：

* 如果一个类加载器收到了类加载的请求，它首先不会自己去尝试加载这个类，而是把请求委托给父加载器去完成，依次向上，因此，所有的类加载请求最终都应该被传递到顶层的启动类加载器中。
* 当不存在父类加载器时，调用启动类加载器完成加载
* 当父加载器/启动类加载器在它的搜索范围中没有找到所需的类时，即无法完成该加载，子加载器才会尝试自己去加载该类。


#### findLoadedClass

检查将要被加载的类是否已经存在。每个类加载器都维护有自己的一份已加载类名字空间，其中不能出现两个同名的类。凡是通过该类加载器加载的类，无论是自己加载的还是委托给其它类加载器加载的，都保存在自己的名字空间中。如果该名字空间中存在指定的类，就返回给类的引用，否则就返回 null。

#### resolveClass

调用此方法，装入指定的类引用的所有其他类。

### 双亲委派的意义

保证系统类在各种类加载器环境中都会被分配给固定的加载器，是同一个类。

## 自定义类加载器

通常情况下，我们都是直接使用系统类加载器。但是，有的时候，我们也需要自定义类加载器。比如应用是通过网络来传输 Java类的字节码。
自定义类加载器一般都是继承自 ClassLoader类，但是为了不破坏双亲委派的模型，更推荐重写 findClass 方法，这样就可以保证新写出的类加载器是符合双亲委派的规则的。本文最后会给出一个实现的例子。

## 破坏双亲委派模型

双亲委派模型并不是一个强制性的约束，双亲委派采用组合而非继承实现，以保证用户可以在需要的时候将父类置为Null而破坏双亲委派模型。
以下行为就会对其进行破坏：

* 重写类加载器loadClass()方法,覆盖双亲委派逻辑
* 父类加载器请求子类去完成加载动作（如SPI）
* 对程序动态性的追求（如代码热替换、热部署）

其中第一个显而易见，我们重点来谈谈另外两点的应用。

### SPI

Java源码中涉及SPI加载的动作，都应该由启动类加载器去加载。  
以JDBC为例，接口Driver定义在rt.jar中，由DriverManager的loadInitialDrivers方法进行加载。代码如下。

```java
private static void loadInitialDrivers() {
        
        ……

        AccessController.doPrivileged(new PrivilegedAction<Void>() {
            public Void run() {

                ServiceLoader<Driver> loadedDrivers = ServiceLoader.load(Driver.class);
                Iterator<Driver> driversIterator = loadedDrivers.iterator();

              
                try{
                    while(driversIterator.hasNext()) {
                        driversIterator.next();
                    }
                } catch(Throwable t) {
                // Do nothing
                }
                return null;
            }
        });

       ……
    }
```
由于Drive接口由用户实现，DriverManager由启动类加载器加载，启动类加载器不“认识“这些由用户实现的类,无法加载。所以我们继续看ServiceLoader类load代码的实现。

```java
    public static <S> ServiceLoader<S> load(Class<S> service) {
        ClassLoader cl = Thread.currentThread().getContextClassLoader();
        return ServiceLoader.load(service, cl);
    }
```
在load（）方法中，调用了线程上下文加载器进行加载。线程上下文类加载器可以由用户自己设置。如果不设置的话，我们继续看方法的调用。


```java

 public static <S> ServiceLoader<S> load(Class<S> service,
                                            ClassLoader loader)
    {
        return new ServiceLoader<>(service, loader);
    }
    
    private ServiceLoader(Class<S> svc, ClassLoader cl) {
        service = Objects.requireNonNull(svc, "Service interface cannot be null");
        loader = (cl == null) ? ClassLoader.getSystemClassLoader() : cl;
        acc = (System.getSecurityManager() != null) ? AccessController.getContext() : null;
        reload();
    }
```

如果没有设置线程上下文加载器，最终通过getSystemClassLoader()获取应用程序类加载器进行加载。而用户实现的代码，默认也是由此加载器进行加载。

### OSGI模块化热部署
TODO

## 实现远程执行功能

	