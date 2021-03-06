---
title: 观察者模式
categories: [Java, DesignPattern]
tags: [design-pattern, java]
---
观察者模式又称订阅-发布模式，是一种一对多的依赖关系，多个观察者对象可同时监听某一主题对象，当该主题对象状态发生变化时，相应的所有观察者对象都可收到通知。

![observer_pattern_uml](https://raw.githubusercontent.com/Leon-WTF/leon-wtf.github.io/master/img/observer_pattern_uml.png)

在getHumidity中会调用notifyObservers来通知所有observers，这里遵循了依赖倒置原则，被观察类（主题）依赖于抽象观察者而非具体观察者。