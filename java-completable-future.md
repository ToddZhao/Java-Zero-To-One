# Day 104: Java高级并发编程 - CompletableFuture高级应用

CompletableFuture是Java 8引入的一个强大的异步编程工具，它不仅提供了Future接口的实现，还提供了函数式编程的特性，使得异步编程变得更加简单和灵活。本文将深入探讨CompletableFuture的高级应用。

## 1. CompletableFuture基础

### 1.1 创建CompletableFuture

```java
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class CompletableFutureBasics {
    public static void main(String[] args) {
        // 创建一个已完成的CompletableFuture
        CompletableFuture<String> completed = CompletableFuture.completedFuture("已完成");
        
        // 创建一个异步执行的CompletableFuture
        CompletableFuture<String> async = CompletableFuture.supplyAsync(() -> {
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
            return "异步执行完成";
        });
        
        // 使用自定义线程池
        ExecutorService executor = Executors.newFixedThreadPool(4);
        CompletableFuture<String> withExecutor = CompletableFuture.supplyAsync(() -> {
            return "使用自定义线程池";
        }, executor);
        
        // 获取结果
        System.out.println(completed.join());
        System.out.println(async.join());
        System.out.println(withExecutor.join());
        
        executor.shutdown();
    }
}
```

### 1.2 转换和组合

```java
public class CompletableFutureTransformation {
    public static void main(String[] args) {
        CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> "Hello")
            // 转换结果
            .thenApply(s -> s + " World")
            // 执行额外操作
            .thenApplyAsync(s -> {
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
                return s.toUpperCase();
            })
            // 消费结果
            .thenAccept(s -> System.out.println("结果是: " + s));
        
        // 等待完成
        future.join();
    }
}
```

## 2. 异常处理

### 2.1 处理异常

```java
public class CompletableFutureExceptionHandling {
    public static void main(String[] args) {
        CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
            if (Math.random() < 0.5) {
                throw new RuntimeException("操作失败");
            }
            return "操作成功";
        })
        .exceptionally(throwable -> {
            System.out.println("发生异常: " + throwable.getMessage());
            return "恢复值";
        })
        .handle((result, throwable) -> {
            if (throwable != null) {
                return "处理异常: " + throwable.getMessage();
            }
            return "处理结果: " + result;
        });
        
        System.out.println(future.join());
    }
}
```

### 2.2 超时处理

```java
import java.util.concurrent.TimeUnit;
import java.util.concurrent.TimeoutException;

public class CompletableFutureTimeout {
    public static void main(String[] args) {
        CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
            try {
                Thread.sleep(2000);
                return "操作完成";
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                return "被中断";
            }
        });
        
        try {
            // 设置超时
            String result = future.orTimeout(1, TimeUnit.SECONDS)
                                 .exceptionally(throwable -> {
                                     if (throwable.getCause() instanceof TimeoutException) {
                                         return "操作超时";
                                     }
                                     return "其他错误";
                                 })
                                 .join();
            System.out.println(result);
        } catch (Exception e) {
            System.out.println("发生异常: " + e.getMessage());
        }
    }
}
```

## 3. 组合多个CompletableFuture

### 3.1 并行执行

```java
public class CompletableFutureComposition {
    public static void main(String[] args) {
        CompletableFuture<String> future1 = CompletableFuture.supplyAsync(() -> {
            sleep(1000);
            return "第一个任务";
        });
        
        CompletableFuture<String> future2 = CompletableFuture.supplyAsync(() -> {
            sleep(2000);
            return "第二个任务";
        });
        
        CompletableFuture<String> future3 = CompletableFuture.supplyAsync(() -> {
            sleep(3000);
            return "第三个任务";
        });
        
        // 等待所有任务完成
        CompletableFuture.allOf(future1, future2, future3)
            .thenRun(() -> {
                System.out.println("所有任务完成");
                System.out.println("结果1: " + future1.join());
                System.out.println("结果2: " + future2.join());
                System.out.println("结果3: " + future3.join());
            })
            .join();
    }
    
    private static void sleep(long millis) {
        try {
            Thread.sleep(millis);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}
```

### 3.2 条件组合

```java
public class CompletableFutureConditional {
    public static void main(String[] args) {
        CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
            sleep(1000);
            return "第一步";
        })
        .thenCompose(result -> {
            // 根据第一步的结果决定下一步操作
            if (result.equals("第一步")) {
                return CompletableFuture.supplyAsync(() -> {
                    sleep(1000);
                    return result + " -> 第二步A";
                });
            } else {
                return CompletableFuture.supplyAsync(() -> {
                    sleep(1000);
                    return result + " -> 第二步B";
                });
            }
        })
        .thenCombine(
            CompletableFuture.supplyAsync(() -> {
                sleep(2000);
                return "并行任务";
            }),
            (result1, result2) -> result1 + " + " + result2
        );
        
        System.out.println(future.join());
    }
    
    private static void sleep(long millis) {
        try {
            Thread.sleep(millis);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}
```

## 4. 实际应用示例

### 4.1 异步HTTP请求

```java
import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;

public class AsyncHttpExample {
    private static final HttpClient client = HttpClient.newHttpClient();
    
    public static void main(String[] args) {
        // 创建多个异步请求
        CompletableFuture<String> weather = getWeather("北京");
        CompletableFuture<String> news = getNews();
        CompletableFuture<String> stockPrice = getStockPrice("AAPL");
        
        // 组合所有请求结果
        CompletableFuture.allOf(weather, news, stockPrice)
            .thenAccept(v -> {
                System.out.println("天气: " + weather.join());
                System.out.println("新闻: " + news.join());
                System.out.println("股票价格: " + stockPrice.join());
            })
            .join();
    }
    
    private static CompletableFuture<String> getWeather(String city) {
        return makeRequest("http://api.weather.com/" + city)
            .exceptionally(throwable -> "获取天气信息失败");
    }
    
    private static CompletableFuture<String> getNews() {
        return makeRequest("http://api.news.com/latest")
            .exceptionally(throwable -> "获取新闻失败");
    }
    
    private static CompletableFuture<String> getStockPrice(String symbol) {
        return makeRequest("http://api.stock.com/price/" + symbol)
            .exceptionally(throwable -> "获取股票价格失败");
    }
    
    private static CompletableFuture<String> makeRequest(String url) {
        HttpRequest request = HttpRequest.newBuilder()
            .uri(URI.create(url))
            .build();
            
        return client.sendAsync(request, HttpResponse.BodyHandlers.ofString())
            .thenApply(HttpResponse::body);
    }
}
```

### 4.2 异步缓存

```java
import java.util.concurrent.ConcurrentHashMap;
import java.util.function.Function;

public class AsyncCache<K, V> {
    private final ConcurrentHashMap<K, CompletableFuture<V>> cache = new ConcurrentHashMap<>();
    private final Function<K, V> computeFunction;
    
    public AsyncCache(Function<K, V> computeFunction) {
        this.computeFunction = computeFunction;
    }
    
    public CompletableFuture<V> get(K key) {
        CompletableFuture<V> future = cache.get(key);
        if (future != null) {
            return future;
        }
        
        CompletableFuture<V> newFuture = new CompletableFuture<>();
        CompletableFuture<V> existingFuture = cache.putIfAbsent(key, newFuture);
        
        if (existingFuture != null) {
            return existingFuture;
        }
        
        // 异步计算值
        CompletableFuture.supplyAsync(() -> {
            try {
                return computeFunction.apply(key);
            } catch (Exception e) {
                cache.remove(key);
                throw e;
            }
        })
        .whenComplete((value, throwable) -> {
            if (throwable != null) {
                newFuture.completeExceptionally(throwable);
            } else {
                newFuture.complete(value);
            }
        });
        
        return newFuture;
    }
    
    public void invalidate(K key) {
        cache.remove(key);
    }
    
    public static void main(String[] args) {
        AsyncCache<String, String> cache = new AsyncCache<>(key -> {
            // 模拟耗时计算
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
            return "Value for " + key;
        });
        
        // 测试异步缓存
        CompletableFuture<String> future1 = cache.get("key1");
        CompletableFuture<String> future2 = cache.get("key1"); // 应该返回相同的Future
        
        System.out.println("是否是同一个Future: " + (future1 == future2));
        System.out.println("值1: " + future1.join());
        System.out.println("值2: " + future2.join());
    }
}
```

## 5. 最佳实践

1. **使用适当的线程池**：为不同类型的任务使用不同的线程池
2. **正确处理异常**：始终添加异常处理逻辑
3. **避免阻塞操作**：在异步操作中避免使用阻塞调用
4. **合理设置超时**：为异步操作设置合理的超时时间
5. **注意资源释放**：及时关闭线程池等资源

## 6. 总结

CompletableFuture是Java中进行异步编程的强大工具，它提供了丰富的API来处理异步操作的转换、组合和异常处理。通过合理使用CompletableFuture，我们可以编写出高效、可维护的异步代码。

在实际应用中，CompletableFuture特别适合处理I/O密集型操作，如网络请求、文件操作等。它的函数式编程特性也使得代码更加简洁和易于理解。

## 参考资源

1. 《Java并发编程实战》
2. 《Java 8实战》
3. Oracle Java文档：https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html
4. Java并发编程网：http://ifeve.com/
5. Modern Java in Action（Manning出版社）