# Day 1: Java多线程编程 - 解放CPU的力量

## 引言

在现代软件开发中，多线程编程已经成为一项不可或缺的技术。随着多核处理器的普及，如何充分利用CPU的多核性能，提高程序的并发处理能力，成为每个Java开发者必须掌握的技能。本文将深入介绍Java多线程编程的核心概念和实践应用。

## 1. 多线程基础

### 1.1 什么是线程

线程是程序执行的最小单位，是进程中的一个实体。一个进程可以包含多个线程，这些线程共享进程的资源。

### 1.2 为什么需要多线程

- 提高CPU利用率
- 增强用户体验
- 充分利用多核处理器
- 提高程序响应速度

### 1.3 创建线程的方式

在Java中，创建线程主要有三种方式：

1. 继承Thread类
```java
public class MyThread extends Thread {
    @Override
    public void run() {
        System.out.println("线程正在运行...");
    }
}

// 使用方式
MyThread thread = new MyThread();
thread.start();
```

2. 实现Runnable接口
```java
public class MyRunnable implements Runnable {
    @Override
    public void run() {
        System.out.println("线程正在运行...");
    }
}

// 使用方式
Thread thread = new Thread(new MyRunnable());
thread.start();
```

3. 使用Callable和Future
```java
public class MyCallable implements Callable<String> {
    @Override
    public String call() throws Exception {
        return "线程执行结果";
    }
}

// 使用方式
FutureTask<String> futureTask = new FutureTask<>(new MyCallable());
Thread thread = new Thread(futureTask);
thread.start();
String result = futureTask.get(); // 获取返回结果
```

## 2. 线程生命周期

线程在Java中有以下几种状态：

- NEW：新创建的线程
- RUNNABLE：可运行/运行中的线程
- BLOCKED：被阻塞的线程
- WAITING：等待状态的线程
- TIMED_WAITING：计时等待的线程
- TERMINATED：终止的线程

```java
public class ThreadLifeCycleDemo {
    public static void main(String[] args) throws InterruptedException {
        Thread thread = new Thread(() -> {
            try {
                Thread.sleep(1000); // TIMED_WAITING状态
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });
        
        System.out.println("初始状态：" + thread.getState()); // NEW
        thread.start();
        System.out.println("启动后状态：" + thread.getState()); // RUNNABLE
        Thread.sleep(500);
        System.out.println("睡眠中状态：" + thread.getState()); // TIMED_WAITING
        thread.join();
        System.out.println("结束后状态：" + thread.getState()); // TERMINATED
    }
}
```

## 3. 线程同步机制

### 3.1 synchronized关键字

```java
public class SynchronizedDemo {
    private int count = 0;
    
    public synchronized void increment() {
        count++;
    }
    
    public synchronized void decrement() {
        count--;
    }
    
    public synchronized int getCount() {
        return count;
    }
}
```

### 3.2 Lock接口

```java
public class LockDemo {
    private final ReentrantLock lock = new ReentrantLock();
    private int count = 0;
    
    public void increment() {
        lock.lock();
        try {
            count++;
        } finally {
            lock.unlock();
        }
    }
}
```

### 3.3 volatile关键字

```java
public class VolatileDemo {
    private volatile boolean flag = false;
    
    public void setFlag() {
        flag = true;
    }
    
    public boolean isFlag() {
        return flag;
    }
}
```

## 4. 线程池

### 4.1 线程池的优势

- 降低资源消耗
- 提高响应速度
- 提高线程的可管理性

### 4.2 常用线程池

```java
public class ThreadPoolDemo {
    public static void main(String[] args) {
        // 固定大小线程池
        ExecutorService fixedPool = Executors.newFixedThreadPool(5);
        
        // 缓存线程池
        ExecutorService cachedPool = Executors.newCachedThreadPool();
        
        // 单线程池
        ExecutorService singlePool = Executors.newSingleThreadExecutor();
        
        // 定时线程池
        ScheduledExecutorService scheduledPool = Executors.newScheduledThreadPool(5);
        
        // 使用示例
        fixedPool.execute(() -> {
            System.out.println("任务正在执行...");
        });
        
        // 关闭线程池
        fixedPool.shutdown();
    }
}
```

### 4.3 自定义线程池

```java
public class CustomThreadPoolDemo {
    public static void main(String[] args) {
        ThreadPoolExecutor executor = new ThreadPoolExecutor(
            2,                      // 核心线程数
            5,                      // 最大线程数
            60L,                    // 空闲线程存活时间
            TimeUnit.SECONDS,       // 时间单位
            new LinkedBlockingQueue<>(10), // 任务队列
            new ThreadPoolExecutor.CallerRunsPolicy() // 拒绝策略
        );
        
        // 提交任务
        executor.execute(() -> {
            System.out.println("自定义线程池任务执行...");
        });
        
        // 关闭线程池
        executor.shutdown();
    }
}
```

## 5. 实践案例

### 5.1 生产者-消费者模式

```java
public class ProducerConsumerDemo {
    private static final int CAPACITY = 10;
    private static BlockingQueue<Integer> queue = new ArrayBlockingQueue<>(CAPACITY);
    
    static class Producer implements Runnable {
        @Override
        public void run() {
            try {
                while (true) {
                    int num = new Random().nextInt(100);
                    queue.put(num);
                    System.out.println("生产者生产数据：" + num);
                    Thread.sleep(100);
                }
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
    
    static class Consumer implements Runnable {
        @Override
        public void run() {
            try {
                while (true) {
                    int num = queue.take();
                    System.out.println("消费者消费数据：" + num);
                    Thread.sleep(200);
                }
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
    
    public static void main(String[] args) {
        ExecutorService service = Executors.newFixedThreadPool(2);
        service.execute(new Producer());
        service.execute(new Consumer());
    }
}
```

### 5.2 并行计算示例

```java
public class ParallelCalculationDemo {
    public static void main(String[] args) throws InterruptedException, ExecutionException {
        int[] numbers = new int[100];
        for (int i = 0; i < numbers.length; i++) {
            numbers[i] = i + 1;
        }
        
        // 创建线程池
        ExecutorService executor = Executors.newFixedThreadPool(4);
        
        // 将数组分成多个部分并行计算和
        int chunkSize = numbers.length / 4;
        List<Future<Integer>> futures = new ArrayList<>();
        
        for (int i = 0; i < 4; i++) {
            final int start = i * chunkSize;
            final int end = (i == 3) ? numbers.length : (i + 1) * chunkSize;
            
            futures.add(executor.submit(() -> {
                int sum = 0;
                for (int j = start; j < end; j++) {
                    sum += numbers[j];
                }
                return sum;
            }));
        }
        
        // 汇总结果
        int totalSum = 0;
        for (Future<Integer> future : futures) {
            totalSum += future.get();
        }
        
        System.out.println("并行计算结果：" + totalSum);
        executor.shutdown();
    }
}
```

## 6. 最佳实践

1. 优先使用实现Runnable接口而不是继承Thread类
2. 正确使用synchronized和Lock
3. 避免过度使用线程
4. 合理使用线程池
5. 注意线程安全问题
6. 使用volatile关键字时要谨慎
7. 遵循线程中断的规范

## 总结

本文介绍了Java多线程编程的核心概念和实践应用，包括：

1. 多线程的基础知识
2. 线程的生命周期
3. 线程同步机制
4. 线程池的使用
5. 实践案例

通过掌握这些知识，我们可以更好地利用多线程技术来提高程序的性能和响应速度。在实际开发中，要根据具体场景选择合适的多线程解决方案，同时要注意避免常见的多线程问题。

## 参考资源

1. Java并发编程实战
2. Java多线程编程核心技术
3. Oracle Java文档：https://docs.oracle.com/javase/tutorial/essential/concurrency/
4. Java并发编程网：http://ifeve.com/