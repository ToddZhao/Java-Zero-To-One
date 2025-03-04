# Day 37: Java并发编程 - Locks和Condition

## 引言

Java除了提供基本的synchronized关键字外，还提供了更加灵活和强大的锁机制。本文将详细介绍Java中的Lock接口及其实现类，以及Condition接口的使用方法。

## 1. Lock接口

### 1.1 概述

Lock接口提供了比synchronized更丰富的功能，包括：
- 非阻塞的获取锁
- 可中断的获取锁
- 超时获取锁
- 公平锁

### 1.2 ReentrantLock示例

```java
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

public class ReentrantLockExample {
    private static final Lock lock = new ReentrantLock();
    private static int count = 0;
    
    public static void increment() {
        lock.lock();
        try {
            count++;
        } finally {
            lock.unlock();
        }
    }
    
    public static void main(String[] args) throws InterruptedException {
        Thread t1 = new Thread(() -> {
            for (int i = 0; i < 1000; i++) {
                increment();
            }
        });
        
        Thread t2 = new Thread(() -> {
            for (int i = 0; i < 1000; i++) {
                increment();
            }
        });
        
        t1.start();
        t2.start();
        t1.join();
        t2.join();
        
        System.out.println("Count: " + count);
    }
}
```

## 2. 读写锁

### 2.1 概述

ReadWriteLock接口允许多个线程同时读取共享资源，但只允许一个线程写入。

### 2.2 ReentrantReadWriteLock示例

```java
import java.util.concurrent.locks.ReadWriteLock;
import java.util.concurrent.locks.ReentrantReadWriteLock;

public class ReadWriteLockExample {
    private static final ReadWriteLock rwLock = new ReentrantReadWriteLock();
    private static String data = "初始数据";
    
    public static String read() {
        rwLock.readLock().lock();
        try {
            return data;
        } finally {
            rwLock.readLock().unlock();
        }
    }
    
    public static void write(String newData) {
        rwLock.writeLock().lock();
        try {
            data = newData;
        } finally {
            rwLock.writeLock().unlock();
        }
    }
    
    public static void main(String[] args) {
        // 创建多个读线程
        for (int i = 0; i < 3; i++) {
            new Thread(() -> {
                System.out.println("读取数据: " + read());
            }).start();
        }
        
        // 创建写线程
        new Thread(() -> {
            write("新数据");
            System.out.println("写入完成");
        }).start();
    }
}
```

## 3. Condition接口

### 3.1 概述

Condition接口提供了类似Object的wait/notify的功能，但是更加灵活。

### 3.2 生产者-消费者示例

```java
import java.util.LinkedList;
import java.util.Queue;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

public class ProducerConsumerExample {
    private static final int CAPACITY = 5;
    private static final Queue<String> queue = new LinkedList<>();
    private static final Lock lock = new ReentrantLock();
    private static final Condition notFull = lock.newCondition();
    private static final Condition notEmpty = lock.newCondition();
    
    public static void produce(String data) throws InterruptedException {
        lock.lock();
        try {
            while (queue.size() == CAPACITY) {
                notFull.await();
            }
            queue.offer(data);
            System.out.println("生产: " + data);
            notEmpty.signal();
        } finally {
            lock.unlock();
        }
    }
    
    public static String consume() throws InterruptedException {
        lock.lock();
        try {
            while (queue.isEmpty()) {
                notEmpty.await();
            }
            String data = queue.poll();
            System.out.println("消费: " + data);
            notFull.signal();
            return data;
        } finally {
            lock.unlock();
        }
    }
    
    public static void main(String[] args) {
        // 生产者线程
        new Thread(() -> {
            try {
                for (int i = 0; i < 10; i++) {
                    produce("数据" + i);
                    Thread.sleep(100);
                }
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }).start();
        
        // 消费者线程
        new Thread(() -> {
            try {
                for (int i = 0; i < 10; i++) {
                    consume();
                    Thread.sleep(200);
                }
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }).start();
    }
}
```

## 4. 最佳实践

1. 锁的使用建议：
   - 优先使用synchronized
   - 需要额外功能时使用Lock
   - 必须在finally块中释放锁
   - 避免死锁

2. 读写锁使用场景：
   - 读多写少的场景
   - 读操作耗时较长
   - 需要保证写操作的原子性

3. Condition使用注意事项：
   - 必须在获取锁的情况下使用
   - 使用while循环检查条件
   - 正确使用signal/signalAll

## 总结

本文介绍了Java并发编程中的高级同步工具：

1. Lock接口及其实现类，提供了比synchronized更灵活的锁机制
2. ReadWriteLock接口，适用于读多写少的场景
3. Condition接口，提供了更强大的线程协作机制

通过合理使用这些工具，我们可以更好地控制多线程程序的同步和协作，提高程序的性能和可靠性。

## 参考资源

1. Java官方文档：https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/locks/package-summary.html
2. Java并发编程实战
3. Java锁机制最佳实践指南
4. Java线程同步性能优化