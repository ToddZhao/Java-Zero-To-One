# Day 101: Java高级并发编程 - 并发设计模式

在Java高级并发编程中，并发设计模式是解决复杂并发问题的成熟方案。这些模式提供了处理多线程环境下常见问题的结构化方法，帮助开发者构建高效、可靠的并发应用。

## 1. 什么是并发设计模式

并发设计模式是针对多线程环境下特定问题的解决方案模板，它们封装了处理线程安全、性能和可维护性的最佳实践。

## 2. 常见的Java并发设计模式

### 2.1 不可变模式（Immutability Pattern）

不可变对象是线程安全的最简单形式，因为它们创建后状态不能被修改。

```java
public final class ImmutableValue {
    private final int value;
    
    public ImmutableValue(int value) {
        this.value = value;
    }
    
    public int getValue() {
        return value;
    }
    
    public ImmutableValue add(int valueToAdd) {
        return new ImmutableValue(this.value + valueToAdd);
    }
}
```

### 2.2 生产者-消费者模式（Producer-Consumer Pattern）

这种模式使用阻塞队列协调生产者和消费者线程。

```java
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.LinkedBlockingQueue;

public class ProducerConsumerExample {
    public static void main(String[] args) {
        BlockingQueue<Integer> queue = new LinkedBlockingQueue<>(10);
        
        // 生产者线程
        Thread producer = new Thread(() -> {
            try {
                for (int i = 0; i < 100; i++) {
                    queue.put(i);
                    System.out.println("生产: " + i);
                    Thread.sleep(100);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });
        
        // 消费者线程
        Thread consumer = new Thread(() -> {
            try {
                while (true) {
                    Integer value = queue.take();
                    System.out.println("消费: " + value);
                    Thread.sleep(200);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });
        
        producer.start();
        consumer.start();
    }
}
```

### 2.3 读-写锁模式（Read-Write Lock Pattern）

允许多个读操作同时进行，但写操作需要独占访问。

```java
import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.locks.ReadWriteLock;
import java.util.concurrent.locks.ReentrantReadWriteLock;

public class ReadWriteMap<K, V> {
    private final Map<K, V> map = new HashMap<>();
    private final ReadWriteLock lock = new ReentrantReadWriteLock();
    
    public V get(K key) {
        lock.readLock().lock();
        try {
            return map.get(key);
        } finally {
            lock.readLock().unlock();
        }
    }
    
    public V put(K key, V value) {
        lock.writeLock().lock();
        try {
            return map.put(key, value);
        } finally {
            lock.writeLock().unlock();
        }
    }
    
    // 其他方法...
}
```

### 2.4 线程池模式（Thread Pool Pattern）

重用线程以减少创建和销毁线程的开销。

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class ThreadPoolExample {
    public static void main(String[] args) {
        ExecutorService executor = Executors.newFixedThreadPool(5);
        
        for (int i = 0; i < 10; i++) {
            final int taskId = i;
            executor.submit(() -> {
                System.out.println("执行任务: " + taskId + " 线程: " + Thread.currentThread().getName());
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            });
        }
        
        executor.shutdown();
    }
}
```

### 2.5 监视器模式（Monitor Object Pattern）

使用synchronized关键字和wait/notify机制实现线程间协作。

```java
public class BoundedBuffer {
    private final Object[] buffer;
    private int putIndex = 0;
    private int takeIndex = 0;
    private int count = 0;
    
    public BoundedBuffer(int capacity) {
        buffer = new Object[capacity];
    }
    
    public synchronized void put(Object x) throws InterruptedException {
        while (count == buffer.length) {
            wait(); // 缓冲区已满，等待
        }
        buffer[putIndex] = x;
        putIndex = (putIndex + 1) % buffer.length;
        count++;
        notifyAll(); // 通知等待的消费者
    }
    
    public synchronized Object take() throws InterruptedException {
        while (count == 0) {
            wait(); // 缓冲区为空，等待
        }
        Object x = buffer[takeIndex];
        takeIndex = (takeIndex + 1) % buffer.length;
        count--;
        notifyAll(); // 通知等待的生产者
        return x;
    }
}
```

### 2.6 双重检查锁定模式（Double-Checked Locking Pattern）

用于延迟初始化单例，减少同步开销。

```java
public class Singleton {
    private static volatile Singleton instance;
    
    private Singleton() {}
    
    public static Singleton getInstance() {
        if (instance == null) { // 第一次检查
            synchronized (Singleton.class) {
                if (instance == null) { // 第二次检查
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

### 2.7 Future模式（Future Pattern）

允许异步计算结果的获取。

```java
import java.util.concurrent.Callable;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;

public class FuturePatternExample {
    public static void main(String[] args) {
        ExecutorService executor = Executors.newSingleThreadExecutor();
        
        // 提交一个耗时计算任务
        Future<Integer> future = executor.submit(new Callable<Integer>() {
            @Override
            public Integer call() throws Exception {
                System.out.println("计算开始...");
                Thread.sleep(3000); // 模拟耗时计算
                return 42;
            }
        });
        
        System.out.println("做其他事情...");
        
        try {
            // 获取计算结果（如果尚未完成会阻塞）
            Integer result = future.get();
            System.out.println("计算结果: " + result);
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            executor.shutdown();
        }
    }
}
```

## 3. 并发设计模式的应用场景

| 设计模式 | 适用场景 | 优点 |
|---------|---------|------|
| 不可变模式 | 共享数据需要被多线程访问但不需要修改 | 简单、安全、无需同步 |
| 生产者-消费者模式 | 数据生成和处理速度不一致 | 解耦生产和消费逻辑，平衡处理速度 |
| 读-写锁模式 | 读操作远多于写操作 | 提高并发读取性能 |
| 线程池模式 | 需要执行大量短期任务 | 减少线程创建开销，控制并发度 |
| 监视器模式 | 需要对共享资源进行互斥访问和线程协作 | 简化同步逻辑 |
| 双重检查锁定 | 延迟初始化单例对象 | 减少同步开销 |
| Future模式 | 需要异步执行任务并在稍后获取结果 | 非阻塞操作，提高响应性 |

## 4. 实际应用示例：并发缓存

下面是一个结合多种并发设计模式的缓存实现：

```java
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.locks.ReadWriteLock;
import java.util.concurrent.locks.ReentrantReadWriteLock;
import java.util.function.Function;

public class ConcurrentCache<K, V> {
    private final ConcurrentHashMap<K, V> cache = new ConcurrentHashMap<>();
    private final ReadWriteLock lock = new ReentrantReadWriteLock();
    private final Function<K, V> computeFunction;
    
    public ConcurrentCache(Function<K, V> computeFunction) {
        this.computeFunction = computeFunction;
    }
    
    public V get(K key) {
        // 首先尝试从缓存获取（无锁）
        V value = cache.get(key);
        if (value != null) {
            return value;
        }
        
        // 缓存未命中，使用读写锁控制计算和更新
        lock.readLock().lock();
        try {
            // 再次检查（可能其他线程已经计算完成）
            value = cache.get(key);
            if (value != null) {
                return value;
            }
        } finally {
            lock.readLock().unlock();
        }
        
        // 需要计算值并更新缓存
        lock.writeLock().lock();
        try {
            // 第三次检查（双重检查模式）
            value = cache.get(key);
            if (value == null) {
                value = computeFunction.apply(key);
                cache.put(key, value);
            }
            return value;
        } finally {
            lock.writeLock().unlock();
        }
    }
    
    public void invalidate(K key) {
        lock.writeLock().lock();
        try {
            cache.remove(key);
        } finally {
            lock.writeLock().unlock();
        }
    }
    
    public void clear() {
        lock.writeLock().lock();
        try {
            cache.clear();
        } finally {
            lock.writeLock().unlock();
        }
    }
}
```

## 5. 最佳实践

1. **选择合适的模式**：根据具体问题选择最适合的并发设计模式。
2. **避免过度设计**：有时简单的synchronized或java.util.concurrent包中的类就足够了。
3. **组合使用**：多种模式可以组合使用以解决复杂问题。
4. **考虑性能影响**：某些模式可能引入额外开销，需要权衡。
5. **测试验证**：使用并发测试工具验证实现的正确性。

## 6. 总结

并发设计模式为解决Java多线程编程中的常见问题提供了成熟的解决方案。掌握这些模式可以帮助开发者构建更加健壮、高效的并发应用。在实际应用中，应根据具体场景选择合适的模式，并注意它们的适用条件和限制。

## 参考资源

1. 《Java并发编程实战》
2. 《七周七并发模型》
3. Oracle Java文档：https://docs.oracle.com/javase/tutorial/essential/concurrency/
4. Java并发编程网：http://ifeve.com/
5. Doug Lea的并发编程著作：http://gee.cs.oswego.edu/dl/cpj/