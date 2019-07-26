---
title: "Java8 Optional与NullPointerException"
category: Java
tag: java
---
[JAVA异常类](https://leon-wtf.github.io/java/2019/06/03/JAVA%E5%BC%82%E5%B8%B8%E7%B1%BB/)列举了Java中部分的异常类，其中最常见的当属NullPointerException了，程序员必须小心提防，所幸Java 8中引入了Optional类这个语法糖来更好的处理这个异常。

比如有如下三个类需要递归引用：
```java
class FirstLayer {
    private SecondLayer secondLayer;
    public SecondLayer getSecondLayer(){
        return secondLayer;
    }
}
class SecondLayer {
    private ThirdLayer thirdLayer;
    public ThirdLayer getThirdLayer() {
        return thirdLayer;
    }
}
class ThirdLayer {
    private String foo;
    public String getFoo(){
        return foo;
    }
}
```
之前的做法是：
```java
FirstLayer firstLayer = new FirstLayer();
if (firstLayer != null && firstLayer.getSecondLayer() != null && firstLayer.getSecondLayer().getThirdLayer() != null) {
    System.out.println(firstLayer.getSecondLayer().getThirdLayer().getFoo());
}
```
现在可以：
```java
Optional.of(new FirstLayer()).map(FirstLayer::getSecondLayer).map(SecondLayer::getThirdLayer).map(ThirdLayer::getFoo).ifPresent(System.out::println);
```
在map函数内部会进行null校验，同时这里还使用了method reference，详细解释请参加：[Java 8 Method Reference: How to Use it](https://www.codementor.io/eh3rrera/using-java-8-method-reference-du10866vx)
甚至还可以：
```java
public static <T> Optional<T> resolve(Supplier<T> resolver) {
    try {
        T result = resolver.get();
        return Optional.ofNullable(result);
    } catch (NullPointerException e){
        return Optional.empty();
    }
}
FirstLayer firstLayer = new FirstLayer();
resolve(() -> firstLayer.getSecondLayer().getThirdLayer().getFoo()).ifPresent(System.out::println);
}
```
其中，Supplier是一种函数式接口(Functional Interface)，就是一个有且仅有一个抽象方法，但是可以有多个非抽象方法的接口。函数式接口可以被实现为anonymous class，更进一步可以转换为lambda表达式，如果只是调用了一个函数，还可以用method reference。
