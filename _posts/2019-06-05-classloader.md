---
title: "类加载器与Class类"
category: Java
tags: [jvm, java]
---
### 类加载器分类 ###
类加载器负责根据类的全限定名将class文件加载到JVM内存，生成Class类的对象。它分为以下几种类型：

1. Bootstrap Classloader
   由C++所写，在JVM启动后初始化，负责加载%JAVA_HOME%/jre/lib，-Xbootclasspath参数指定的路径以及%JAVA_HOME%/jre/classes中的类
2. ExtClassLoader
   是sun.misc.Launcher的内部类，继承自java.net.URLClassLoader->java.security.SecureClassLoader->java.lang.ClassLoader,在rt.jar中，由Bootstrap Classloader加载，负责加载%JAVA_HOME%/jre/lib/ext和java.ext.dirs系统变量指定路径中的类。parent ClassLoader为null（因为Bootstrap Classloader并不是由java实现的）。
3. AppClassLoader
   也是sun.misc.Launcher的内部类，继承自java.net.URLClassLoader，负责加载来自在命令java中的-classpath或者java.class.path系统属性或者CLASSPATH系统属性所指定的路径中的类。其parent ClassLoader为ExtClassLoader，且是我们自定义类默认的类加载器。
###类加载过程###
双亲委派机制：如果一个类未加载，那么必须先由其父加载器（Bootstrap Classloader可以认为是ExtClassLoader父加载器）尝试加载，如果父加载器在其路径内找不到该类才由子加载器加载。可以防止核心类被外来类覆盖。具体的源码分析可以参见[深入理解Java类加载器(ClassLoader)](https://blog.csdn.net/javazejian/article/details/73413292)。
下面是结合源码，画出的利用AppClassLoader查找类的流程图：

![jvm_class_loader_workflow](https://raw.githubusercontent.com/Leon-WTF/leon-wtf.github.io/master/img/jvm_class_loader_workflow.png)

### Class类 ###
我们通常写的用class（首字母c小写）定义的类，表征了java虚拟机里对象的类型（java是强类型语言），但同时这些类又都是java.lang.Class(首字母C大写)的对象，通过AppClassLoader加载进虚拟机内存方法区。每个类都对应一个独一无二的Class对象，包括Java基本类型、void关键字及数组（所有同一维度和类型的数组拥有同样的Class，数组的长度不做考虑。对应Class的名字表示为维度和类型。比如一个整型数据的Class名为“[I”，字节型三维数组Class名为“[[[B”，两维对象数组Class名为“[[Ljava.lang.Object”）。得到Class对象的方法有三种：
```
MyObject foo = New MyObject();
Class c = foo.getClass();

Class c = Class.ForName("MyObject");

Class c = MyObject.class;
```