# Day 12: Java并发工具类

## 引言

Java提供了丰富的并发工具类，这些工具类可以帮助我们更好地处理多线程编程中的各种场景。本文将详细介绍Java并发工具类的核心概念和实践应用。

## 1. 同步工具类

### 1.1 CountDownLatch

```java
public class CountDownLatchExample {
    public static void main(String[] args) throws InterruptedException {
        int threadCount = 3;
        CountDownLatch latch = new CountDownLatch(threadCount);
        
        for (int i = 0; i < threadCount; i++) {
            new Thread(() -> {
                try {
                    System.out.println(Thread.currentThread().getName() + " 开始执行");
                    Thread.sleep(1000);
                    latch.countDown();
                    System.out.println(Thread.currentThread().getName() + " 执行完成");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }).start();
        }
        
        latch.await();
        System.out.println("所有线程执行完成");
    }
}
```

### 1.2 CyclicBarrier

```java
public class CyclicBarrierExample {
    public static void main(String[] args) {
        int threadCount = 3;
        CyclicBarrier barrier = new CyclicBarrier(threadCount, () -> 
            System.out.println("所有线程到达屏障点"));
        
        for (int i = 0; i < threadCount; i++) {
            new Thread(() -> {
                try {
                    System.out.println(Thread.currentThread().getName() + " 准备就绪");
                    barrier.await();
                    System.out.println(Thread.currentThread().getName() + " 继续执行");
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }).start();
        }
    }
}
```

### 1.3 Semaphore

```java
public class SemaphoreExample {
    public static void main(String[] args) {
        int permits = 3;
        Semaphore semaphore = new Semaphore(permits);
        
        for (int i = 0; i < 10; i++) {
            new Thread(() -> {
                try {
                    semaphore.acquire();
                    System.out.println(Thread.currentThread().getName() + " 获得许可");
                    Thread.sleep(1000);
                    System.out.println(Thread.currentThread().getName() + " 释放许可");
                    semaphore.release();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }).start();
        }
    }
}
```

## 2. 并发集合

### 2.1 ConcurrentHashMap

```java
public class ConcurrentMapExample {
    private static final ConcurrentHashMap<String, Integer> map = 
        new ConcurrentHashMap<>();
    
    public static void main(String[] args) throws InterruptedException {
        Thread t1 = new Thread(() -> {
            for (int i = 0; i < 1000; i++) {
                map.put("key" + i, i);
            }
        });
        
        Thread t2 = new Thread(() -> {
            for (int i = 0; i < 1000; i++) {
                map.put("key" + i, i + 1);
            }
        });
        
        t1.start();
        t2.start();
        t1.join();
        t2.join();
        
        System.out.println("Map size: " + map.size());
    }
}
```

### 2.2 ConcurrentLinkedQueue

```java
public class ConcurrentQueueExample {
    private static final ConcurrentLinkedQueue<String> queue = 
        new ConcurrentLinkedQueue<>();
    
    public static void main(String[] args) {
        // 生产者线程
        new Thread(() -> {
            for (int i = 0; i < 10; i++) {
                queue.offer("Item " + i);
                System.out.println("生产: Item " + i);
                try {
                    Thread.sleep(100);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }).start();
        
        // 消费者线程
        new Thread(() -> {
            while (true) {
                String item = queue.poll();
                if (item != null) {
                    System.out.println("消费: " + item);
                }
            }
        }).start();
    }
}
```

## 3. 线程安全的数据结构

### 3.1 AtomicInteger

```java
public class AtomicExample {
    private static final AtomicInteger counter = new AtomicInteger(0);
    
    public static void main(String[] args) throws InterruptedException {
        Thread[] threads = new Thread[10];
        
        for (int i = 0; i < threads.length; i++) {
            threads[i] = new Thread(() -> {
                for (int j = 0; j < 1000; j++) {
                    counter.incrementAndGet();
                }
            });
            threads[i].start();
        }
        
        for (Thread thread : threads) {
            thread.join();
        }
        
        System.out.println("Final count: " + counter.get());
    }
}
```

### 3.2 CopyOnWriteArrayList

```java
public class CopyOnWriteExample {
    private static final CopyOnWriteArrayList<String> list = 
        new CopyOnWriteArrayList<>();
    
    public static void main(String[] args) {
        // 写线程
        new Thread(() -> {
            for (int i = 0; i < 10; i++) {
                list.add("Item " + i);
                System.out.println("添加: Item " + i);
                try {
                    Thread.sleep(100);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }).start();
        
        // 读线程
        new Thread(() -> {
            while (true) {
                for (String item : list) {
                    System.out.println("读取: " + item);
                }
            }
        }).start();
    }
}
```

## 4. 实践案例

### 4.1 线程安全的缓存

```java
public class ThreadSafeCache<K, V> {
    private final ConcurrentHashMap<K, V> cache = new ConcurrentHashMap<>();
    private final Semaphore semaphore = new Semaphore(10); // 限制并发访问数
    
    public V get(K key, Supplier<V> valueLoader) {
        V value = cache.get(key);
        if (value != null) {
            return value;
        }
        
        try {
            semaphore.acquire();
            try {
                value = cache.get(key);
                if (value == null) {
                    value = valueLoader.get();
                    cache.put(key, value);
                }
                return value;
            } finally {
                semaphore.release();
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new RuntimeException(e);
        }
    }
    
    public void invalidate(K key) {
        cache.remove(key);
    }
}
```

### 4.2 并发任务执行器

```java
public class ConcurrentTaskExecutor<T> {
    private final BlockingQueue<Runnable> taskQueue;
    private final List<Thread> workers;
    private final AtomicBoolean isRunning;
    
    public ConcurrentTaskExecutor(int nThreads) {
        this.taskQueue = new LinkedBlockingQueue<>();
        this.workers = new ArrayList<>(nThreads);
        this.isRunning = new AtomicBoolean(true);
        
        for (int i = 0; i < nThreads; i++) {
            Thread worker = new Thread(() -> {
                while (isRunning.get() || !taskQueue.isEmpty()) {
                    try {
                        Runnable task = taskQueue.poll(100, TimeUnit.MILLISECONDS);
                        if (task != null) {
                            task.run();
                        }
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                        break;
                    }
                }
            });
            workers.add(worker);
            worker.start();
        }
    }
    
    public void submit(Runnable task) {
        if (isRunning.get()) {
            taskQueue.offer(task);
        }
    }
    
    public void shutdown() {
        isRunning.set(false);
        workers.forEach(Thread::interrupt);
    }
}
```

## 5. 最佳实践

1. 选择合适的并发工具类
2. 正确处理中断异常
3. 避免过度同步
4. 合理使用线程安全集合
5. 注意性能和内存开销

## 总结

本文介绍了Java并发工具类的核心概念和实践应用，包括：

1. 同步工具类的使用
2. 并发集合的应用
3. 线程安全的数据结构
4. 实践案例
5. 最佳实践建议

通过掌握这些工具类，我们可以更好地处理多线程编程中的各种场景，提高程序的性能和可靠性。

## 参考资源

1. Java官方文档：https://docs.oracle.com/javase/tutorial/essential/concurrency/
2. Java并发编程实战
3. 并发工具类最佳实践指南
4. Java性能优化建议