---
title: "Java concurrency in practice"
category: Java
tag: java
---
# Fundamentals
- Atomicity
Race condition: The correctness of a computation depends on the relative timing or interleaving of multiple threads by the runtime. Ex：Check-And-Act, a potentially stale observation is used to make a decision on what to do next

- Visibility
Stale data: See an out-of-date value
Writes and reads of volatile long and double values are always atomic.
Writes to and reads of references are always atomic, regardless of whether they are implemented as 32-bit or 64-bit values.
"volatile" keyword can ensure the visibility and can be used in below cases:  
	- Writes to the variable do not depend on its current value, or you can ensure that only a single thread ever updates the value
	- The variable does not participate in invariants with other state variables
	- Locking is not required for any other reason while the variable is being accessed.
- Publish
Making the object available to code outside of its current scope, such as by storing a reference to it where other code can find it, returning it from a non-private method, or passing it to a method in another class
- Escape
An object that is published when it should not have been
```java
public class ThisEscape {
	public final String name;
	public ThisEscape(EventSource source) {
		source.registerListener(
			new EventListener() {
				public void onEvent(Event e) {
					System.out.println("name: "+ThisEscape.this.name);
				}
			});
		this.name = "test";
	}
}
```
When ThisEscape publishes the EventListener, it implicitly publishes the enclosing ThisEscape instance as well, because inner class instances contain a hidden reference to the enclosing instance. But an object is in a predictable, consistent state only after its constructor returns, so publishing an object from within its constructor can publish an incompletely constructed object。Equally，start a new thread in the constructor containing the "this" reference variable，will also let the this reference escape. Using a private constructor and a public factory method can avoid this：
```java
public class SafeListener {
	//Blank final, can be initialized in the constructor only.
	private final EventListener listener;
	public final String name;
	private SafeListener() {
		listener = new EventListener() {
			public void onEvent(Event e) {
				System.out.println("name: "+ThisEscape.this.name);
			}
		};
		this.name = "test";
	}
	public static SafeListener newInstance(EventSource source) {
		SafeListener safe = new SafeListener();
		source.registerListener(safe.listener);
		return safe;
	}
}
```
- Thread confinement
	Stack confinement: Limit the object access by local variable, so only in one thread
	ThreadLocal: [ThreadPool实现原理](https://leon-wtf.github.io/leon.github.io/java/2019/06/26/threadpool/)
- Immutability and safe publish
The final field is a means of what is sometimes called safe publication . Here, "publication" of an object means creating it in one thread and then having that newly-created object be referred to by another thread at some point in the future. When the JVM executes the constructor of your object, it must store values into the various fields of the object, and store a pointer to the object data. As in any other case of data writes, these accesses can potentially occur out of order, and their application to main memory can be delayed and other processors can be delayed unless you take special steps to combat this. In particular, the pointer to the object data could be stored to main memory and accessed before the fields themselves have been committed (this can happen partly because of compiler ordering: if you think about how you'd write things in a low-level language such as C or assembler, it's quite natural to store a pointer to a block of memory, and then advance the pointer as you're writing data to that block). And this in turn could lead to another thread seeing the object in an invalid or partially constructed state.
final prevents this from happening: if a field is final , it is part of the JVM specification that it must effectively ensure that, once the object pointer is available to other threads, so are the correct values of that object's final fields.
A properly constructed object can be safely published by:
• Initializing an object reference from a static initializer(Static initializers are run by the JVM at class initialization time, after class loading but before the class is used by any thread. Because the JVM acquires a lock during initialization [JLS 12.4.2] and this lock is acquired by each thread at least once to ensure that the class has been loaded, memory writes made during static initialization are automatically visible to all threads.)
• Storing a reference to it into a volatile field or AtomicReference
• Storing a reference to it into a final field of a properly constructed object
• Storing a reference to it into a field that is properly guarded by a lock(including thread  safe container)
[final关键字深入解析](https://juejin.im/post/5b8821b5e51d4538a108c969)
- Happens-before
	- Program order rule. Each action in a thread happens-before every action in that thread that comes later in the program order.
	- Monitor lock rule. An unlock on a monitor lock happens-before every subsequent lock on that same monitor lock.
	- Volatile variable rule. A write to a volatile field happens-before every subsequent read of that same field.
	- Thread start rule. A call to Thread.start on a thread happens-before every action in the started thread.
	- Thread termination rule. Any action in a thread happens-before any other thread detects that thread has terminated, either by successfully return from Thread.join or by Thread.isAlive returning false.
	- Interruption rule. A thread calling interrupt on another thread happens-before the interrupted thread detects the interrupt (either by having InterruptedException thrown, or invoking isInterrupted or interrupted).
	- Finalizer rule. The end of a constructor for an object happens-before the start of the finalizer for that object.
	- Transitivity. If A happens-before B, and B happens-before C, then A happens-before C.
 
> [Java concurrency in practice](https://github.com/Leon-WTF/leon.github.io/blob/master/doc/java-concurrency-in-practice.pdf)

# AtomaticInteger #
Use volatile value and unsafe.getAndAddInt(this, valueOffset, 1) to ensue both atomicity and visibility.
```java
public final int getAndAddInt(Object var1, long var2, int var4) {
    int var5;
    do {
        var5 = this.getIntVolatile(var1, var2);
        // Compare And Set:基于硬件的原子操作支持
    } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));
    return var5;
}
```
# Thread Life-circle # 
![thread-life-circle](https://img-blog.csdnimg.cn/20190802173319535.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjkwOTA1NQ==,size_16,color_FFFFFF,t_70)
[Life Cycle of a Thread in Java](https://www.baeldung.com/java-thread-lifecycle)
# FutureTask #
**FutureTask**实现了**runable**接口，在**run**函数里调用传入的**callable**函数。其内部维护了函数运行的状态，当有另外的线程调用其**get**函数时将线程加入内部的waiters链表，然后调用**park**函数进行等待，当函数运行完成时调用**unpark**唤醒所有等待线程。
[java并发编程之FutureTask](https://www.jianshu.com/p/06f8df545a86)
# Synchronized #
**synchronized**关键字在字节码中对应的是用**monitorenter**和**monitorexit**将同步代码包裹起来，对于同步方法是添加**ACC_SYNCHRONIZED**标志。JVM为每一个对象关联一个**ObjectMonitor**对象，对应的会有**enter**和**exist**函数，用来实现针对这个对象的监视器锁。**synchronized**如果每次都直接调用**enter**和**exist**函数, 都会用到**park/unpark**，其底层是用操作系统**mutex**和**condition**实现，涉及到用户态和系统态直接的切换，需要花费很多的处理器时间，所以后面JVM又提出了自旋锁、偏向锁、轻量级锁等优化

[Java虚拟机是如何执行线程同步的](http://www.hollischuang.com/archives/1876)
[Synchronized的实现原理](https://www.hollischuang.com/archives/1883)
[Moniter的实现原理](https://www.hollischuang.com/archives/2030)
[JVM系列-（二）OOP-KLASS](https://zhuanlan.zhihu.com/p/51695160)
[谈谈LockSupport](https://benjaminwhx.com/2018/05/01/%E3%80%90%E7%BB%86%E8%B0%88Java%E5%B9%B6%E5%8F%91%E3%80%91%E8%B0%88%E8%B0%88LockSupport/)
[JVM源码分析之synchronized实现](https://www.jianshu.com/p/c5058b6fe8e5)

# ReentrantLock #
基于[AbstractQueuedSynchronizer](https://www.cnblogs.com/waterystone/p/4920797.html)实现，内部分为公平锁和非公平锁。公平锁满足FIFO机制，非公平锁允许后来的线程抢占锁。
