---
title: "Spring AOP vs AspectJ"
category: SpringBoot
tag: aop
---
## 概述
这两种都是目前流行的实现面向切面编程(Aspect Oriented Programming)的框架，AOP基本概念和详细的比较可以参见：[Comparing Spring AOP and AspectJ](https://www.baeldung.com/spring-aop-vs-aspectj)
1. Spring AOP是基于代理方法的运行时织入，可以分为JDK代理和CGLIB代理，详情可以参见：[动态代理模式](https://leon-wtf.github.io/java/designpattern/2019/06/21/dynamic-proxy-pattern/)
2. AspectJ是借助特殊的编译器直接将切面织入生成的二进制代码中

## 比较
Spring AOP使用方便，但功能简单，只能实现方法执行时的织入，其有诸多限制，比如：
1. JDK代理是基于接口的，所以要求被代理的类实现了接口
2. CGLIB代理是基于类继承的，所以要求被代理的类和方法不能是static、final和private的
3. 由于是用新生成的代理类去替换原来的类，所以，在被代理的类内部之间的方法调用是使用的原来的方法而不是新生成的代理类中的方法
4. 仅可用于被Spring容器管理的类
AspectJ使用复杂，但功能强大，可以实现任意位置的织入，且没有上面的诸多限制

## 使用
下面讲一下如何使用AspectJ在Spring Boot工程中实现AOP:
首先，基本的配置如下：
[Intro to AspectJ](https://www.baeldung.com/aspectj)
目前工程中使用的是post-compile weaving方式, 可以不需要引入aspectjweaver，同时通过weaveDirectories指定了被织入class文件位置。
```
<weaveDirectories>
    <weaveDirectory>${project.build.directory}/classes</weaveDirectory>
</weaveDirectories>
```
其次，要想利用AspectJ模式的事务还需要以下配置：
1. 在工程入口类上指定事务管理器模式：
```
@EnableTransactionManagement(mode=AdviceMode.ASPECTJ)
```
2. 在工程依赖和plugin依赖中都引入spring-aspects
[Spring @Transactional With AspectJ](http://sevenlist.github.io/2014/08/24/spring-at-transactional-with-aspectj/)
这是因为被AspectJ织入的aspect（实现事务的代码）是二进制形式的，在spring-aspects中，而不是源代码形式的：
[Applying already compiled aspect JARs](http://www.mojohaus.org/aspectj-maven-plugin/examples/libraryJars.html)
