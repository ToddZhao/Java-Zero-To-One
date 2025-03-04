# Day 38: Java线程池和Executor框架

## 引言

在Java并发编程中，线程池是一个非常重要的概念。通过线程池，我们可以复用线程，避免频繁创建和销毁线程带来的开销，提高系统的性能。本文将详细介绍Java线程池的使用方法和Executor框架。

## 1. 线程池基础

### 1.1 为什么需要线程池

- 减少资源消耗：重复利用已创建的线程
- 提高响应速度：任务到达时，无需创建线程即可执行
- 提高线程的可管理性：统一管理线程的创建和销毁

### 1.2 线程池的核心参数

```java
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler)
```

- corePoolSize：核心线程数
- maximumPoolSize：最大线程数
- keepAliveTime：线程空闲时间
- workQueue：工作队列
- threadFactory：线程工厂
- handler：拒绝策略

## 2. 常用线程池

### 2.1 FixedThreadPool

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class FixedThreadPoolExample {
    public static void main(String[] args) {
        // 创建固定大小的线程池
        ExecutorService executor = Executors.newFixedThreadPool(3);
        
        // 提交任务
        for (int i = 0; i < 10; i++) {
            final int taskId = i;
            executor.execute(() -> {
                System.out.println("Task " + taskId + " is running on " + 
                    Thread.currentThread().getName());
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            });
        }
        
        // 关闭线程池
        executor.shutdown();
    }
}
```

### 2.2 CachedThreadPool

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class CachedThreadPoolExample {
    public static void main(String[] args) {
        // 创建缓存线程池
        ExecutorService executor = Executors.newCachedThreadPool();
        
        // 提交任务
        for (int i = 0; i < 10; i++) {
            final int taskId = i;
            executor.execute(() -> {
                System.out.println("Task " + taskId + " is running on " + 
                    Thread.currentThread().getName());
            });
        }
        
        executor.shutdown();
    }
}
```

## 3. 自定义线程池

### 3.1 ThreadPoolExecutor示例

```java
import java.util.concurrent.*;

public class CustomThreadPoolExample {
    public static void main(String[] args) {
        // 创建自定义线程池
        ThreadPoolExecutor executor = new ThreadPoolExecutor(
            2,                      // 核心线程数
            4,                      // 最大线程数
            60L,                    // 空闲线程存活时间
            TimeUnit.SECONDS,       // 时间单位
            new LinkedBlockingQueue<Runnable>(10),  // 工作队列
            Executors.defaultThreadFactory(),       // 线程工厂
            new ThreadPoolExecutor.CallerRunsPolicy()  // 拒绝策略
        );
        
        // 提交任务
        for (int i = 0; i < 20; i++) {
            final int taskId = i;
            executor.execute(() -> {
                System.out.println("Task " + taskId + " is running on " + 
                    Thread.currentThread().getName());
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            });
        }
        
        executor.shutdown();
    }
}
```

## 4. 提交任务和获取结果

### 4.1 Future接口

```java
import java.util.concurrent.*;

public class FutureExample {
    public static void main(String[] args) throws Exception {
        ExecutorService executor = Executors.newFixedThreadPool(1);
        
        // 提交有返回值的任务
        Future<Integer> future = executor.submit(() -> {
            Thread.sleep(1000);
            return 42;
        });
        
        // 获取任务结果
        System.out.println("任务结果: " + future.get());
        
        executor.shutdown();
    }
}
```

### 4.2 CompletableFuture

```java
import java.util.concurrent.CompletableFuture;

public class CompletableFutureExample {
    public static void main(String[] args) {
        CompletableFuture<String> future = CompletableFuture
            .supplyAsync(() -> "Hello")
            .thenApply(s -> s + " World")
            .thenApply(String::toUpperCase);
        
        System.out.println(future.join());
    }
}
```

## 5. 最佳实践

1. 线程池使用建议：
   - 根据任务类型选择合适的线程池
   - 避免使用无界队列
   - 合理设置线程池参数
   - 及时关闭线程池

2. 任务提交策略：
   - 优先使用execute提交无返回值任务
   - 需要返回值时使用submit
   - 灵活使用CompletableFuture

3. 异常处理：
   - 在任务中妥善处理异常
   - 使用Future时注意处理中断异常
   - 实现UncaughtExceptionHandler

## 总结

本文介绍了Java线程池和Executor框架的核心内容：

1. 线程池的基本概念和核心参数
2. 常用的线程池类型及其使用场景
3. 自定义线程池的创建和配置
4. 任务提交和结果获取的方式

通过合理使用线程池，我们可以更好地管理线程资源，提高程序的性能和可靠性。

## 参考资源

1. Java官方文档：https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ThreadPoolExecutor.html
2. Java并发编程实战
3. 线程池最佳实践指南
4. Java性能优化实践