---
title: "Java异常类"
category: Java
tag: java
---
Java常见异常类UML图如下：

![java_throwable](https://raw.githubusercontent.com/Leon-WTF/leon-wtf.github.io/master/img/java_throwable.png)

- Error通常为虚拟机相关错误，通常比较严重，除了通知用户和尽力使程序安全终止之外，紧靠应用自身无法恢复，所以应用程序不应对其捕获，。Exception可以在程序内进行捕获并处理，使应用程序继续正常运行。
- Exception又分RuntimeException，通常由程序员编写代码错误导致，非RuntimeException，通常为应用程序运行环境中的错误导致，编译器会检测是否有try-catch处理或在方法签名处有throws关键字，否则无法通过编译，所以又称checked Exception。