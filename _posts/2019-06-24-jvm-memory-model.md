---
title: "JVM内存模型"
category: Java
tags: [jvm, java]
---
![jvm_memory_model](https://raw.githubusercontent.com/Leon-WTF/leon-wtf.github.io/master/img/jvm_memory_model.png)

从JDK8起, Permanent Generation区被移除，Runtime Constant Pool(除了Symbolic References)和Static Field被移到了Heap区，Symbolic References和Class Metadata（类型信息：完整有效名，直接父类完整有效名，修饰符，直接接口有序表，域信息：域名，类型，修饰符，方法信息：方法名，返回类型，参数类型，修饰符，代码，局部变量区大小，异常表，类加载器）被移到了JVM管理的内存之外的Native Memory，叫做Metaspace区。JVM通过装载、连接和初始化使该类型可以被正在运行的java程序所使用，类被加载后生成java.lang.Class的对象存在Heap区。
String.intern函数的行为也有所改变，如果在字符串常量池（在堆内存）中找到该字符串，则返回字符串常量池（保存首次出现的字面量字符串）内的对象引用，如果没有找到，则在字符串常量池中保存当前字符串的引用，并返回当前字符串的引用。
```java
/**
 * Returns a canonical representation for the string object.
 * <p>
 * A pool of strings, initially empty, is maintained privately by the
 * class {@code String}.
 * <p>
 * When the intern method is invoked, if the pool already contains a
 * string equal to this {@code String} object as determined by
 * the {@link #equals(Object)} method, then the string from the pool is
 * returned. Otherwise, this {@code String} object is added to the
 * pool and a reference to this {@code String} object is returned.
 * <p>
 * It follows that for any two strings {@code s} and {@code t},
 * {@code s.intern() == t.intern()} is {@code true}
 * if and only if {@code s.equals(t)} is {@code true}.
 * <p>
 * All literal strings and string-valued constant expressions are
 * interned. String literals are defined in section 3.10.5 of the
 * <cite>The Java&trade; Language Specification</cite>.
 *
 * @return  a string that has the same contents as this string, but is
 *          guaranteed to be from a pool of unique strings.
 */
public native String intern();
```