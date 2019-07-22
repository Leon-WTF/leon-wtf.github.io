---
title: "Thread and AbstractQueuedSynchronizer"
category: Java
tags: [concurrency, java]
---
Thread中join实现如下：

```java
public final synchronized void join(long millis) throws InterruptedException {
    long base = System.currentTimeMillis();
    long now = 0;

    if (millis < 0) {
        throw new IllegalArgumentException("timeout value is negative");
    }

    if (millis == 0) {
        while (isAlive()) {
            wait(0);
        }
    } else {
        while (isAlive()) {
            long delay = millis - now;
            if (delay <= 0) {
                break;
            }
            wait(delay);
            now = System.currentTimeMillis() - base;
        }
    }
}
```
其中利用了Object的wait方法，调用的前提是已经获得join线程的锁，如果thread对象被锁住则会等待其被释放。
```java
import core.AbstractCommonConsumer;
import core.CommonUtils;
import core.MagnetMonitor;
import core.NamedThreadPoolExecutor;
import org.apache.ibatis.io.Resources;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStreamReader;
import java.lang.reflect.InvocationTargetException;
import java.util.Date;
import java.util.LinkedList;
import java.util.List;
import java.util.concurrent.*;
import java.util.concurrent.locks.LockSupport;

class JoinTester01 implements Runnable {

    private String name;

    JoinTester01(String name) {
        this.name = name;
    }

    public void run() {
        System.out.printf("%s begins: %s\n", name, new Date());
        try {
            TimeUnit.SECONDS.sleep(10);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.printf("%s has finished: %s\n", name, new Date());
    }
}
class JoinTester02 implements Runnable {

    Thread thread;

    JoinTester02(Thread thread) {
        this.thread = thread;
    }

    Thread getThread(){
        return this.thread;
    }

    public void run() {
        synchronized (thread) {
            System.out.printf("getObjectLock at %s\n", new Date());
            try {
                Thread.sleep(2000);
            } catch (InterruptedException ex) {
                ex.printStackTrace();
            }
            System.out.printf("ReleaseObjectLock at %s\n", new Date());
        }
        try {
            Thread.sleep(1000);
        } catch (InterruptedException ex) {
            ex.printStackTrace();
        }
        synchronized (thread) {
            System.out.printf("getObjectLock again at %s\n", new Date());
            try {
                Thread.sleep(2000);
            } catch (InterruptedException ex) {
                ex.printStackTrace();
            }
            System.out.printf("ReleaseObjectLock again at %s\n", new Date());
        }
    }
}
class TestMain {
    public static void main(String[] args) {
        Thread thread = new Thread(new JoinTester01("Leon"));
        JoinTester02 tester02 = new JoinTester02(thread);
        Thread getLockThread = new Thread(tester02);

        getLockThread.start();
        thread.start();
        try {
            System.out.printf("start join at %s\n", new Date());
            thread.join(1000);
            System.out.printf("stop join at %s\n", new Date());
        } catch (InterruptedException e) {
            // TODO Auto-generated catch block
            e.printStackTrace();
        }
    }
}
```
```
start join at Sun Jun 30 18:40:08 CST 2019
Leon begins: Sun Jun 30 18:40:08 CST 2019
getObjectLock at Sun Jun 30 18:40:08 CST 2019
ReleaseObjectLock at Sun Jun 30 18:40:10 CST 2019
stop join at Sun Jun 30 18:40:11 CST 2019
getObjectLock again at Sun Jun 30 18:40:11 CST 2019
ReleaseObjectLock again at Sun Jun 30 18:40:13 CST 2019
Leon has finished: Sun Jun 30 18:40:18 CST 2019
```

[Thread详解](https://www.cnblogs.com/waterystone/p/4920007.html)
[Java并发之AQS详解](https://www.cnblogs.com/waterystone/p/4920797.html)