---
title: "装饰模式 vs (静态)代理模式"
category: Java
tags: [design-pattern, java]
---
这两个设计模式都属于结构型模式，且非常相似，其UML图如下：
装饰模式：

![decorative_pattern_uml](https://raw.githubusercontent.com/Leon-WTF/leon.github.io/master/img/decorative_pattern_uml.png)
如下IO方法就是使用了装饰模式：
```
BufferedReader reader = new BufferedReader(new InputStreamReader(Resources.getResourceAsStream(resource))
```
（静态）代理模式：提到代理模式一般是指静态代理模式，动态代理模式会在[动态代理实现原理](https://segmentfault.com/a/1190000019355525)中专门讲解

![static_proxy_pattern_uml](https://raw.githubusercontent.com/Leon-WTF/leon.github.io/master/img/static_proxy_pattern_uml.png)

共同点：
- 装饰者与被装饰者，代理类与被代理类，都是继承自同一个接口，可以令他们在被调用时相互替换

不同点：
- 被装饰者往往被作为装饰者的构造器参数传入装饰者，强调被装饰者功能的增强；被代理类往往在代理类内部被创建，所以这里用UML里组合的关系，强调对被代理类的访问控制。
- 装饰者里持有的是被装饰者的接口类型，所以可以装饰所有实现同一接口的类；代理类是针对某一个具体的类进行代理，所以对每一个类都要实现一个对应的代理类，这是静态代理模式的局限，可以使用动态代理模式来弥补。