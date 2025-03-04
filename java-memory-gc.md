# Day 13: Java内存管理与垃圾回收

## 引言

Java的内存管理和垃圾回收机制是其最重要的特性之一，它让开发者不需要手动管理内存，从而提高了开发效率和程序的健壮性。本文将深入介绍Java内存管理和垃圾回收的核心概念和实践应用。

## 1. JVM内存结构

### 1.1 内存区域划分

JVM内存主要分为以下几个区域：

1. 堆区（Heap）
   - 新生代（Young Generation）
     - Eden区
     - Survivor区（From和To）
   - 老年代（Old Generation）

2. 方法区（Method Area）
   - 存储类信息、常量、静态变量等

3. 程序计数器（Program Counter Register）
   - 当前线程执行的字节码位置

4. 虚拟机栈（VM Stack）
   - 存储局部变量表、操作数栈等

5. 本地方法栈（Native Method Stack）
   - 执行本地方法时使用

## 2. 垃圾回收机制

### 2.1 垃圾判定算法

```java
// 1. 引用计数法示例
public class ReferenceCountingExample {
    public static void main(String[] args) {
        Object obj1 = new Object();
        Object obj2 = new Object();
        
        obj1 = obj2; // obj1原来引用的对象变为垃圾
        obj1 = null; // obj2原来引用的对象变为垃圾
    }
}

// 2. 可达性分析示例
public class GCRootExample {
    private static Object staticObj;    // 静态变量作为GC Root
    private Object instanceObj;         // 实例变量不是GC Root
    
    public void method() {
        Object localObj = new Object(); // 局部变量可能作为GC Root
    }
}
```

### 2.2 垃圾回收算法

1. 标记-清除算法（Mark-Sweep）
2. 复制算法（Copying）
3. 标记-整理算法（Mark-Compact）
4. 分代收集算法（Generational Collection）

## 3. 垃圾收集器

### 3.1 常见的垃圾收集器

```java
// 垃圾收集器配置示例
public class GCConfiguration {
    public static void main(String[] args) {
        // 使用G1收集器
        // -XX:+UseG1GC
        
        // 使用并行收集器
        // -XX:+UseParallelGC
        
        // 使用CMS收集器
        // -XX:+UseConcMarkSweepGC
        
        // 设置堆大小
        // -Xms4g -Xmx4g
    }
}
```

### 3.2 GC调优参数

```java
public class GCTuning {
    public static void main(String[] args) {
        // 设置新生代大小
        // -Xmn1g
        
        // 设置survivor比例
        // -XX:SurvivorRatio=8
        
        // 设置老年代与新生代比例
        // -XX:NewRatio=2
        
        // 设置垃圾收集器线程数
        // -XX:ParallelGCThreads=4
    }
}
```

## 4. 内存泄漏问题

### 4.1 常见的内存泄漏场景

```java
// 1. 集合类泄漏
public class CollectionLeakExample {
    private static List<Object> list = new ArrayList<>();
    
    public void addItem(Object item) {
        list.add(item);  // 只添加不删除导致内存泄漏
    }
}

// 2. 监听器泄漏
public class ListenerLeakExample {
    private List<EventListener> listeners = new ArrayList<>();
    
    public void addEventListener(EventListener listener) {
        listeners.add(listener);
    }
    
    // 忘记实现removeEventListener方法
}

// 3. 资源未关闭
public class ResourceLeakExample {
    public static void readFile(String fileName) {
        FileInputStream fis = null;
        try {
            fis = new FileInputStream(fileName);
            // 处理文件
        } catch (IOException e) {
            e.printStackTrace();
        }
        // 忘记关闭流
    }
}
```

### 4.2 内存泄漏检测

```java
public class MemoryLeakDetection {
    public static void main(String[] args) {
        // 启用内存泄漏检测
        // -XX:+HeapDumpOnOutOfMemoryError
        // -XX:HeapDumpPath=/path/to/dump.hprof
        
        // 使用jmap生成堆转储
        // jmap -dump:format=b,file=heap.bin <pid>
        
        // 使用jhat分析堆转储
        // jhat heap.bin
    }
}
```

## 5. 实践案例

### 5.1 内存监控

```java
public class MemoryMonitor {
    public static void printMemoryInfo() {
        Runtime runtime = Runtime.getRuntime();
        
        long totalMemory = runtime.totalMemory();
        long freeMemory = runtime.freeMemory();
        long maxMemory = runtime.maxMemory();
        
        System.out.println("总内存：" + totalMemory / 1024 / 1024 + "MB");
        System.out.println("空闲内存：" + freeMemory / 1024 / 1024 + "MB");
        System.out.println("最大内存：" + maxMemory / 1024 / 1024 + "MB");
    }
    
    public static void main(String[] args) {
        printMemoryInfo();
        System.gc(); // 建议进行垃圾回收
        printMemoryInfo();
    }
}
```

### 5.2 弱引用使用

```java
public class WeakReferenceExample {
    public static void main(String[] args) {
        // 创建弱引用
        WeakReference<Object> weakRef = new WeakReference<>(new Object());
        
        // 检查引用是否还有效
        Object obj = weakRef.get();
        if (obj != null) {
            System.out.println("对象仍然存活");
        } else {
            System.out.println("对象已被回收");
        }
        
        // 强制垃圾回收
        System.gc();
        
        // 再次检查
        obj = weakRef.get();
        if (obj != null) {
            System.out.println("对象仍然存活");
        } else {
            System.out.println("对象已被回收");
        }
    }
}
```

## 6. 最佳实践

1. 合理设置堆大小
2. 选择适合的垃圾收集器
3. 及时释放不用的对象
4. 避免内存泄漏
5. 定期监控内存使用情况

## 总结

本文介绍了Java内存管理与垃圾回收的核心概念和实践应用，包括：

1. JVM内存结构
2. 垃圾回收机制
3. 垃圾收集器类型
4. 内存泄漏问题
5. 实践案例和最佳实践

通过掌握这些知识，我们可以更好地理解Java的内存管理机制，编写出更高效、更可靠的程序。

## 参考资源

1. Java虚拟机规范
2. 深入理解Java虚拟机
3. Java性能优化实践
4. GC调优指南