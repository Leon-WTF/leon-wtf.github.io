---
title: "Java Monitor（管程）"
category: Java
tags: [java, concurrency]
---
操作系统在面对线程间同步的时候，会支持例如semaphore信号量和mutex互斥量等同步原语，而monitor是在编程语言中被实现的，下面介绍一下java中monitor（监视器/管程：管理共享变量以及对其的操作过程，让他们支持并发）的实现原理：

![monitor_model](https://raw.githubusercontent.com/Leon-WTF/leon-wtf.github.io/master/img/monitor_model.png)

以一个阻塞队列的实现来举例：
![blocked_queue](https://raw.githubusercontent.com/Leon-WTF/leon-wtf.github.io/master/img/blocked_queue.png)

同时，java内置的synchronized关键字可以认为是MESA模型的简化版，其只能有一个条件变量，但编译器会自动添加加锁与解锁的代码。synchronized关键字可以修饰实例方法、类方法以及代码块，如果修饰的是代码块，需要制定关联的Object；如果修饰的是实例方法，那么其关联的对象实际上是this；如果修饰的是类方法，那么其关联的对象是this.class。这些关联的对象就是MESA模型里的条件变量。
###synchronized实现原理###
JVM基于进入和退出monitor对象来实现同步，同步代码块采用添加moniterenter、moniterexit，同步方法使用ACC_SYNCHRONIZED标记符隐式实现。每个对象都有一个monitor与之关联，运行到moniterenter时尝试获取对应monitor的所有权，获取成功就将monitor的进入数加1（所以是可重入锁，也被称为重量级锁），否则就阻塞，拥有monitor的线程运行到moniterexit时进入数减1，为0时释放monitor。
java中每个对象都有一个对象头，synchronized所用的锁就是存在对象头里的。如果是非数组的对象是8个字节（32位JVM）或者16字节（64位JVM），数组对象还会有一个数组长度（4个字节）。以32位JVM非数组对象为例：

![java_object_header](https://raw.githubusercontent.com/Leon-WTF/leon-wtf.github.io/master/img/java_object_header.png)

锁信息就存在前4个字节的MarkWord中，JVM对synchronized的加锁过程优化为：
1. 检测Mark Word里面是不是当前线程的ID，如果是，表示当前线程处于偏向锁 
2. 如果不是，则使用CAS将当前线程的ID替换Mard Word，如果成功则表示当前线程获得偏向锁，置偏向标志位1 
3. 如果失败，则说明发生竞争，撤销偏向锁，进而升级为轻量级锁。 
4. 当前线程使用CAS将对象头的Mark Word替换为锁记录指针，如果成功，当前线程获得锁 
5. 如果失败，表示其他线程竞争锁，当前线程便尝试使用自旋来获取锁。 
6. 如果自旋成功则依然处于轻量级状态。 
7. 如果自旋失败，则升级为重量级锁。
重量级锁是悲观锁的一种，自旋锁、轻量级锁与偏向锁属于乐观锁。
CAS设计读取-比较-写入三个操作，是在CPU指令层面保证其原子性，volatile是保证多线程下的内存可见性，二者需配合使用。另外还需注意CPU缓存行（一次以32/64字节为单位从主内存中读取数据到缓存）包含多个变量所带来的隐形同步问题：其中一个变量被volatile修饰，导致另外一个变量在另一个CPU核上（另一个线程）的读写也要被强制刷新缓存。

> [管程：并发编程的万能钥匙](https://time.geekbang.org/column/article/1fa80f363c15ddb62f7b20595357a01f/share?code=fCn1Hz13YaRxOIk9j7BVRdH132emx%2FLoqY9ZwEV2c7Y%3D&oss_token=96c8fdfef2257b85)
> [java 中的锁 -- 偏向锁、轻量级锁、自旋锁、重量级锁](https://blog.csdn.net/zqz_zqz/article/details/70233767)