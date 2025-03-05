# Day 103: Java高级并发编程 - Fork/Join框架

Fork/Join框架是Java 7引入的一种用于并行执行任务的框架，它是ExecutorService接口的一种实现，专门针对可以递归分解的问题设计。Fork/Join框架基于"工作窃取"算法，能够充分利用多核处理器的优势，提高并行计算的效率。

## 1. Fork/Join框架概述

Fork/Join框架的核心思想是"分而治之"（Divide and Conquer）：将一个大任务分解成多个小任务并行执行，然后合并这些小任务的结果。

### 1.1 Fork/Join框架的特点

- **工作窃取算法**：空闲的工作线程可以从其他忙碌线程的队列中窃取任务
- **适合递归分解的问题**：如归并排序、快速排序、矩阵乘法等
- **自动负载均衡**：任务会被均匀分配到所有工作线程
- **充分利用多核处理器**：提高CPU利用率

### 1.2 Fork/Join框架的核心组件

- **ForkJoinPool**：执行ForkJoinTask的线程池
- **ForkJoinTask**：可以在ForkJoinPool中执行的任务基类
- **RecursiveTask**：有返回结果的ForkJoinTask
- **RecursiveAction**：没有返回结果的ForkJoinTask

## 2. 使用Fork/Join框架

### 2.1 创建ForkJoinPool

```java
import java.util.concurrent.ForkJoinPool;

public class ForkJoinPoolExample {
    public static void main(String[] args) {
        // 创建使用所有可用处理器的ForkJoinPool
        ForkJoinPool commonPool = ForkJoinPool.commonPool();
        System.out.println("公共池的并行度: " + commonPool.getParallelism());
        
        // 创建自定义并行度的ForkJoinPool
        ForkJoinPool customPool = new ForkJoinPool(4);
        System.out.println("自定义池的并行度: " + customPool.getParallelism());
    }
}
```

### 2.2 实现RecursiveTask

下面是一个使用Fork/Join框架计算数组元素和的例子：

```java
import java.util.concurrent.ForkJoinPool;
import java.util.concurrent.RecursiveTask;
import java.util.Arrays;

public class ArraySumCalculator extends RecursiveTask<Long> {
    private final int[] array;
    private final int start;
    private final int end;
    private static final int THRESHOLD = 1000; // 阈值，决定何时停止任务分解
    
    public ArraySumCalculator(int[] array, int start, int end) {
        this.array = array;
        this.start = start;
        this.end = end;
    }
    
    @Override
    protected Long compute() {
        int length = end - start;
        
        // 如果任务足够小，直接计算
        if (length <= THRESHOLD) {
            return computeDirectly();
        }
        
        // 否则，将任务一分为二
        int middle = start + length / 2;
        
        ArraySumCalculator leftTask = new ArraySumCalculator(array, start, middle);
        ArraySumCalculator rightTask = new ArraySumCalculator(array, middle, end);
        
        // 异步执行左半部分
        leftTask.fork();
        
        // 直接执行右半部分
        Long rightResult = rightTask.compute();
        
        // 等待左半部分结果并合并
        Long leftResult = leftTask.join();
        
        return leftResult + rightResult;
    }
    
    private long computeDirectly() {
        long sum = 0;
        for (int i = start; i < end; i++) {
            sum += array[i];
        }
        return sum;
    }
    
    public static void main(String[] args) {
        // 创建一个包含1百万个元素的数组
        int[] array = new int[1_000_000];
        Arrays.fill(array, 1); // 填充数组，每个元素值为1
        
        ForkJoinPool pool = ForkJoinPool.commonPool();
        
        // 创建计算任务
        ArraySumCalculator calculator = new ArraySumCalculator(array, 0, array.length);
        
        // 执行任务并获取结果
        long sum = pool.invoke(calculator);
        
        System.out.println("数组元素总和: " + sum);
    }
}
```

### 2.3 实现RecursiveAction

下面是一个使用Fork/Join框架对数组进行并行排序的例子：

```java
import java.util.Arrays;
import java.util.concurrent.ForkJoinPool;
import java.util.concurrent.RecursiveAction;

public class ParallelMergeSortAction extends RecursiveAction {
    private final int[] array;
    private final int[] temp;
    private final int start;
    private final int end;
    private static final int THRESHOLD = 1000;
    
    public ParallelMergeSortAction(int[] array, int[] temp, int start, int end) {
        this.array = array;
        this.temp = temp;
        this.start = start;
        this.end = end;
    }
    
    @Override
    protected void compute() {
        int length = end - start;
        
        // 如果任务足够小，直接排序
        if (length <= THRESHOLD) {
            Arrays.sort(array, start, end);
            return;
        }
        
        // 否则，将任务一分为二
        int middle = start + length / 2;
        
        // 创建子任务
        ParallelMergeSortAction leftTask = new ParallelMergeSortAction(array, temp, start, middle);
        ParallelMergeSortAction rightTask = new ParallelMergeSortAction(array, temp, middle, end);
        
        // 并行执行子任务
        invokeAll(leftTask, rightTask);
        
        // 合并结果
        merge(start, middle, end);
    }
    
    private void merge(int start, int middle, int end) {
        // 复制到临时数组
        System.arraycopy(array, start, temp, start, end - start);
        
        int i = start;
        int j = middle;
        int k = start;
        
        // 合并两个已排序的子数组
        while (i < middle && j < end) {
            if (temp[i] <= temp[j]) {
                array[k++] = temp[i++];
            } else {
                array[k++] = temp[j++];
            }
        }
        
        // 复制剩余元素
        while (i < middle) {
            array[k++] = temp[i++];
        }
    }
    
    public static void main(String[] args) {
        // 创建一个随机数组
        int[] array = new int[1_000_000];
        for (int i = 0; i < array.length; i++) {
            array[i] = (int) (Math.random() * 1000000);
        }
        
        int[] temp = new int[array.length];
        
        ForkJoinPool pool = ForkJoinPool.commonPool();
        
        // 创建排序任务
        ParallelMergeSortAction sortTask = new ParallelMergeSortAction(array, temp, 0, array.length);
        
        // 执行任务
        pool.invoke(sortTask);
        
        // 验证排序结果
        boolean sorted = true;
        for (int i = 1; i < array.length; i++) {
            if (array[i-1] > array[i]) {
                sorted = false;
                break;
            }
        }
        
        System.out.println("数组是否已排序: " + sorted);
    }
}
```

## 3. Fork/Join框架的工作原理

### 3.1 工作窃取算法

Fork/Join框架的核心是工作窃取（Work-Stealing）算法：

1. 每个工作线程都有自己的双端队列（deque）来存储任务
2. 当一个线程的队列为空时，它会从其他忙碌线程的队列尾部窃取任务
3. 线程总是从自己队列的头部获取任务，从其他线程队列的尾部窃取任务
4. 这种机制能够自动平衡工作负载，减少线程空闲时间

### 3.2 任务分割策略

有效使用Fork/Join框架的关键是选择合适的任务分割策略：

1. **阈值选择**：任务分割的阈值不能太小（会导致过多的任务创建和调度开销），也不能太大（会导致并行度不足）
2. **平衡分割**：尽量使子任务的工作量均衡
3. **避免过度分割**：考虑硬件的并行能力，避免创建过多的小任务

## 4. 高级应用示例

### 4.1 并行文件搜索

```java
import java.io.File;
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.ForkJoinPool;
import java.util.concurrent.RecursiveTask;

public class ParallelFileSearch extends RecursiveTask<List<File>> {
    private final File directory;
    private final String fileExtension;
    
    public ParallelFileSearch(File directory, String fileExtension) {
        this.directory = directory;
        this.fileExtension = fileExtension;
    }
    
    @Override
    protected List<File> compute() {
        List<File> results = new ArrayList<>();
        List<ParallelFileSearch> subTasks = new ArrayList<>();
        
        // 获取目录内容
        File[] files = directory.listFiles();
        if (files != null) {
            for (File file : files) {
                if (file.isDirectory()) {
                    // 为子目录创建新任务
                    ParallelFileSearch subTask = new ParallelFileSearch(file, fileExtension);
                    subTask.fork();
                    subTasks.add(subTask);
                } else {
                    // 检查文件扩展名
                    if (file.getName().endsWith(fileExtension)) {
                        results.add(file);
                    }
                }
            }
        }
        
        // 收集所有子任务的结果
        for (ParallelFileSearch subTask : subTasks) {
            results.addAll(subTask.join());
        }
        
        return results;
    }
    
    public static void main(String[] args) {
        ForkJoinPool pool = ForkJoinPool.commonPool();
        
        // 从根目录开始搜索所有.java文件
        ParallelFileSearch searchTask = new ParallelFileSearch(new File("/path/to/search"), ".java");
        
        List<File> javaFiles = pool.invoke(searchTask);
        
        System.out.println("找到 " + javaFiles.size() + " 个Java文件");
        for (File file : javaFiles) {
            System.out.println(file.getPath());
        }
    }
}
```

### 4.2 并行图像处理

```java
import java.awt.image.BufferedImage;
import java.io.File;
import java.io.IOException;
import javax.imageio.ImageIO;
import java.util.concurrent.ForkJoinPool;
import java.util.concurrent.RecursiveAction;

public class ParallelImageProcessor extends RecursiveAction {
    private final BufferedImage image;
    private final int startX, startY, endX, endY;
    private static final int THRESHOLD = 100000; // 像素数阈值
    
    public ParallelImageProcessor(BufferedImage image, int startX, int startY, int endX, int endY) {
        this.image = image;
        this.startX = startX;
        this.startY = startY;
        this.endX = endX;
        this.endY = endY;
    }
    
    @Override
    protected void compute() {
        int width = endX - startX;
        int height = endY - startY;
        int pixelCount = width * height;
        
        if (pixelCount <= THRESHOLD) {
            // 直接处理像素
            processPixels(startX, startY, endX, endY);
            return;
        }
        
        // 将图像区域分为四个子区域
        int midX = startX + width / 2;
        int midY = startY + height / 2;
        
        // 创建四个子任务
        invokeAll(
            new ParallelImageProcessor(image, startX, startY, midX, midY),
            new ParallelImageProcessor(image, midX, startY, endX, midY),
            new ParallelImageProcessor(image, startX, midY, midX, endY),
            new ParallelImageProcessor(image, midX, midY, endX, endY)
        );
    }
    
    private void processPixels(int startX, int startY, int endX, int endY) {
        // 这里实现具体的像素处理逻辑，例如灰度转换
        for (int y = startY; y < endY; y++) {
            for (int x = startX; x < endX; x++) {
                int rgb = image.getRGB(x, y);
                
                // 提取RGB分量
                int red = (rgb >> 16) & 0xFF;
                int green = (rgb >> 8) & 0xFF;
                int blue = rgb & 0xFF;
                
                // 计算灰度值
                int gray = (red + green + blue) / 3;
                
                // 创建新的灰度像素
                int newPixel = (gray << 16) | (gray << 8) | gray;
                
                // 设置新的像素值
                image.setRGB(x, y, newPixel);
            }
        }
    }
    
    public static void main(String[] args) {
        try {
            // 加载图像
            BufferedImage image = ImageIO.read(new File("input.jpg"));
            
            // 创建并执行任务
            ForkJoinPool pool = ForkJoinPool.commonPool();
            ParallelImageProcessor processor = new ParallelImageProcessor(
                image, 0, 0, image.getWidth(), image.getHeight());
            pool.invoke(processor);
            
            // 保存处理后的图像
            ImageIO.write(image, "jpg", new File("output_grayscale.jpg"));
            System.out.println("图像处理完成");
            
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

## 5. Fork/Join框架的最佳实践

1. **选择合适的阈值**：根据任务特性和硬件环境调整阈值，以平衡任务分割的粒度
2. **避免任务中的同步**：尽量避免在任务中使用同步，这会降低并行性能
3. **避免任务间的依赖**：子任务应该尽可能独立，减少等待和协调开销
4. **合理使用fork()和compute()**：对于左右子任务，通常一个使用fork()异步执行，另一个直接使用compute()在当前线程执行
5. **注意join()的位置**：在所有fork()操作之后再调用join()，避免不必要的阻塞

## 6. Fork/Join框架与并行流的关系

Java 8引入的并行流（Parallel Streams）在内部使用Fork/Join框架实现并行处理：

```java
import java.util.Arrays;
import java.util.List;

public class ParallelStreamExample {
    public static void main(String[] args) {
        List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);
        
        // 使用并行流计算和
        long sum = numbers.parallelStream()
                          .mapToLong(i -> i)
                          .sum();
        
        System.out.println("Sum: " + sum);
        
        // 使用并行流进行过滤和转换
        List<Integer> evenSquares = numbers.parallelStream()
                                         .filter(n -> n % 2 == 0)
                                         .map(n -> n * n)
                                         .toList();
        
        System.out.println("Even squares: " + evenSquares);
    }
}
```

## 7. 总结

Fork/Join框架是Java并发编程工具箱中的强大工具，特别适合于可以递归分解的计算密集型任务。它通过工作窃取算法实现了高效的负载均衡，能够充分利用多核处理器的计算能力。

在实际应用中，开发者需要根据任务特性选择合适的任务分割策略和阈值，以获得最佳性能。对于简单的并行处理需求，可以考虑使用Java 8引入的并行流，它在内部使用了Fork/Join框架，但提供了更简洁的API。

## 参考资源

1. 《Java并发编程实战》
2. 《Java 8实战》
3. Oracle Java文档：https://docs.oracle.com/javase/tutorial/essential/concurrency/forkjoin.html
4. Doug Lea的Fork/Join框架论文：http://gee.cs.oswego.edu/dl/papers/fj.pdf