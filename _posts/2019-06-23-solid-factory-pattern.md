---
title: "SOLID设计原则与工厂模式"
categories: [OOD, DesignPattern] 
tags: [ood, design-pattern]
---
在面向对象设计领域里，SOLID是非常经典的设计原则，可以认为它是道，设计模式是术，只有深刻理解了道，才能用好术。下面简单总结一下SOLID原则：

- Single Responsibility Principle: 每个类只能有一个被修改的原因
- Open-Close Principle: 对扩展开发，对修改关闭
- Liskov's Substitution Principle: 派生类必须能够完全替换基类[Liskov's Substitution Principle(LSP)](https://www.oodesign.com/liskov-s-substitution-principle.html)
- Interface Segregation Principle：客户端不应该被强制依赖他们不需要使用的接口
- Dependency Inversion Principle: 高层次的模块不应该依赖低层次的模块, 双方都应该依赖抽象。抽象不应该依赖具体细节。细节应该依赖抽象。[Dependency Inversion Principle](https://www.oodesign.com/dependency-inversion-principle.html)

下面以工厂模式为例，说明一下SOLID原则在设计模式里的体现：
工厂模式属于创建型模式，主要分三种：
- 简单工厂模式
- 工厂方法模式
- 抽象工厂模式
个人觉得第三种模式使用场景较少且比较鸡肋，主要介绍前两种。
先来看下简单工厂模式：

![simple_factory_uml](https://raw.githubusercontent.com/Leon-WTF/leon-wtf.github.io/master/img/simple_factory_uml.png)

```java
public abstract class Operation{
    private double value1;
    private double value2;

    public double getValue1() {
        return value1;
    }

    public void setValue1(double value1) {
        this.value1 = value1;
    }

    public double getValue2() {
        return value2;
    }

    public void setValue2(double value2) {
        this.value2 = value2;
    }
    
    protected abstract double getResult();
}
public class OperationAdd extends Operation {
    @Override
    protected double getResult(){
        return getValue1() + getValue2();
    }
}
public class OperationMinus extends Operation {
    @Override
    protected double getResult(){
        return getValue1() - getValue2();
    }
}
public class OperationMul extends Operation {
    @Override
    protected double getResult(){
        return getValue1() * getValue2();
    }
}
public class OperationFactory{
    public static Operation createOperation(String operation){
        Operation operation = null;
        switch(operation){
            case "+":
                operation = new OperationAdd();
                break;
            case "-":
                operation = new OperationMinus();
                break;
            case "*":
                operation = new OperationMul();
                break;
            default:
                throw new UnsupportedOperationException("Unsupported Operation:" + operation);
        }
        return operation;
    }
}
```
首先，我们必须令Operation的派生类遵循Liskov's Substitution Principle，才能放心的说，无论我们在工厂中创建出哪种Operation的派生类，都能够利用多态替换其后对Operation的引用。
其次，工厂模式返回抽象类，使调用工厂的高层模块依赖Operation这个抽象类而不是其某个具体的派生类，这满足了Dependency Inversion Principle。
但是，OperationFactory类中包含了所有Operation派生类的创建，后面如果不断的需要增加新的Operation派生类，就需要不断的修改OperationFactory，这违反了Open-Close Principle，就需要引入工厂方法模式：

![abstract_factory_uml](https://raw.githubusercontent.com/Leon-WTF/leon-wtf.github.io/master/img/abstract_factory_uml.png)

```java
public interface IFactory {
    Operation CreateOption();
}
public class AddFactory implements IFactory {
    public Operation CreateOption() {
        return new OperationAdd();
    }
}
public class MulFactory implements IFactory {
    public Operation CreateOption() {
        return new OperationMul();
    }
}
public class SubFactory implements IFactory {
    public Operation CreateOption() {
        return new OperationSub();
    }
}
```
这样每当有新的Operation派生类，只需要对应新建新的工厂类就可以了，这其实也是将工厂类与其调用者用抽象层隔离了。但要注意这也会因为创建过多的类而难以管理。