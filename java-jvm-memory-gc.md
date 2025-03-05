# Day 106: JVM进阶 - JVM内存模型与垃圾回收机制

JVM内存模型和垃圾回收机制是Java开发者必须深入理解的核心知识。掌握这些知识不仅有助于编写高效的代码，还能帮助我们解决内存泄漏、性能瓶颈等问题，对于构建高性能、高可用的Java应用至关重要。

## 1. JVM内存结构

JVM内存结构在Java 8及以后的版本中有了较大变化，主要分为以下几个区域：

### 1.1 堆（Heap）

堆是JVM中最大的一块内存区域，被所有线程共享，主要用于存储对象实例和数组。

```java
public class HeapDemo {
    public static void main(String[] args) {
        // 获取JVM堆的最大内存
        long maxMemory = Runtime.getRuntime().maxMemory();
        // 获取JVM堆的总内存
        long totalMemory = Runtime.getRuntime().totalMemory();
        // 获取JVM堆的空闲内存
        long freeMemory = Runtime.getRuntime().freeMemory();
        
        System.out.println("最大内存: " + maxMemory / 1024 / 1024 + "MB");
        System.out.println("总内存: " + totalMemory / 1024 / 1024 + "MB");
        System.out.println("空闲内存: " + freeMemory / 1024 / 1024 + "MB");
    }
}
```

堆内存分为年轻代（Young Generation）和老年代（Old Generation）：

- **年轻代**：又分为Eden区和两个Survivor区（From和To）
- **老年代**：存放长时间存活的对象

### 1.2 方法区（Method Area）

方法区用于存储已被JVM加载的类信息、常量、静态变量、即时编译器编译后的代码等数据。在Java 8之前，方法区也被称为永久代（PermGen），Java 8及以后被元空间（Metaspace）取代。

```java
public class MethodAreaDemo {
    // 静态变量存储在方法区
    private static final String CONSTANT = "Hello, Method Area!";
    
    public static void main(String[] args) {
        // 类信息、方法信息等存储在方法区
        Class<?> clazz = MethodAreaDemo.class;
        System.out.println("类名: " + clazz.getName());
        System.out.println("常量: " + CONSTANT);
    }
}
```

### 1.3 程序计数器（Program Counter Register）

程序计数器是一块较小的内存空间，它可以看作是当前线程所执行的字节码的行号指示器。

### 1.4 虚拟机栈（VM Stack）

虚拟机栈描述的是Java方法执行的内存模型：每个方法在执行的同时会创建一个栈帧，用于存储局部变量表、操作数栈、动态链接、方法出口等信息。

```java
public class StackDemo {
    public static void main(String[] args) {
        method1();
    }
    
    public static void method1() {
        int a = 10; // 局部变量存储在栈中
        method2();
    }
    
    public static void method2() {
        int b = 20; // 局部变量存储在栈中
        method3();
    }
    
    public static void method3() {
        int c = 30; // 局部变量存储在栈中
        // 获取当前线程的栈跟踪
        StackTraceElement[] stackTrace = Thread.currentThread().getStackTrace();
        for (StackTraceElement element : stackTrace) {
            System.out.println(element);
        }
    }
}
```

### 1.5 本地方法栈（Native Method Stack）

本地方法栈与虚拟机栈类似，区别在于虚拟机栈为Java方法服务，而本地方法栈则为Native方法服务。

## 2. 垃圾回收算法

### 2.1 标记-清除算法（Mark-Sweep）

标记-清除算法分为两个阶段：
1. 标记阶段：标记所有需要回收的对象
2. 清除阶段：回收被标记的对象所占用的空间

**缺点**：效率低，会产生大量内存碎片

### 2.2 复制算法（Copying）

复制算法将内存分为两块，每次只使用其中一块。当这一块内存用完了，就将还存活的对象复制到另一块上面，然后再把已使用过的内存空间一次清理掉。

**优点**：实现简单，运行高效
**缺点**：内存利用率低，只有一半的内存可用

### 2.3 标记-整理算法（Mark-Compact）

标记-整理算法在标记-清除算法的基础上，将所有存活的对象都向一端移动，然后直接清理掉端边界以外的内存。

**优点**：解决了内存碎片问题
**缺点**：移动对象需要额外的时间开销

### 2.4 分代收集算法（Generational Collection）

分代收集算法根据对象的存活周期将内存划分为几块，不同的区域采用不同的垃圾回收算法：

- **年轻代**：使用复制算法
- **老年代**：使用标记-清除或标记-整理算法

## 3. 垃圾收集器

### 3.1 Serial收集器

Serial收集器是最基本、历史最悠久的垃圾收集器，它是一个单线程收集器，在进行垃圾收集时，必须暂停所有的工作线程，直到垃圾收集结束。

**适用场景**：客户端应用，内存较小的场景

### 3.2 ParNew收集器

ParNew收集器是Serial收集器的多线程版本，除了使用多线程进行垃圾收集外，其余行为与Serial收集器完全一样。

**适用场景**：多CPU环境下的服务端应用

### 3.3 Parallel Scavenge收集器

Parallel Scavenge收集器是一个新生代收集器，它也是使用复制算法的多线程收集器，但它的目标是达到一个可控制的吞吐量。

**适用场景**：注重吞吐量的应用，如后台计算型应用

### 3.4 CMS收集器（Concurrent Mark Sweep）

CMS收集器是一种以获取最短回收停顿时间为目标的收集器，它基于标记-清除算法实现，整个过程分为四个步骤：

1. 初始标记（CMS initial mark）
2. 并发标记（CMS concurrent mark）
3. 重新标记（CMS remark）
4. 并发清除（CMS concurrent sweep）

**优点**：并发收集，低停顿
**缺点**：对CPU资源敏感，无法处理浮动垃圾，会产生内存碎片

### 3.5 G1收集器（Garbage-First）

G1收集器是一款面向服务端应用的垃圾收集器，它将堆内存分割成多个大小相等的独立区域（Region），并且保留了分代的概念。

**特点**：
- 并行与并发
- 分代收集
- 空间整合
- 可预测的停顿

```java
public class G1Demo {
    public static void main(String[] args) {
        // 使用G1收集器的JVM参数
        // -XX:+UseG1GC
        // -XX:MaxGCPauseMillis=200
        // -XX:InitiatingHeapOccupancyPercent=45
        
        // 创建大量对象以触发GC
        List<byte[]> list = new ArrayList<>();
        while (true) {
            byte[] bytes = new byte[1024 * 1024]; // 1MB
            list.add(bytes);
            try {
                Thread.sleep(10);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```

### 3.6 ZGC收集器（Z Garbage Collector）

ZGC是JDK 11中推出的一款低延迟垃圾收集器，它的目标是在任何堆大小下都能将STW（Stop-The-World）时间控制在10ms以内。

**特点**：
- 并发
- 基于Region
- 压缩
- 低延迟

## 4. GC调优实践

### 4.1 常用JVM参数

```
# 堆内存设置
-Xms<size>        # 初始堆大小
-Xmx<size>        # 最大堆大小
-Xmn<size>        # 年轻代大小

# 垃圾收集器设置
-XX:+UseSerialGC           # 使用Serial + Serial Old收集器
-XX:+UseParallelGC         # 使用Parallel Scavenge + Parallel Old收集器
-XX:+UseConcMarkSweepGC    # 使用ParNew + CMS收集器
-XX:+UseG1GC               # 使用G1收集器
-XX:+UseZGC                # 使用ZGC收集器

# GC日志设置
-XX:+PrintGCDetails        # 打印GC详细信息
-XX:+PrintGCTimeStamps     # 打印GC时间戳
-Xloggc:<file>             # 指定GC日志文件
```

### 4.2 内存泄漏排查

```java
public class MemoryLeakDemo {
    private static List<Object> leakList = new ArrayList<>();
    
    public static void main(String[] args) {
        // 模拟内存泄漏
        while (true) {
            leakList.add(new byte[1024 * 1024]); // 添加1MB的对象
            try {
                Thread.sleep(100);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```

排查内存泄漏的工具：

1. **jstat**：监视虚拟机各种运行状态信息
   ```
   jstat -gcutil <pid> 1000 10
   ```

2. **jmap**：生成堆转储快照
   ```
   jmap -dump:format=b,file=heap.bin <pid>
   ```

3. **jhat**：分析堆转储快照
   ```
   jhat heap.bin
   ```

4. **VisualVM**：可视化工具，用于监控、分析Java应用程序

### 4.3 GC调优案例

#### 案例1：频繁Full GC问题

**症状**：应用响应时间变长，频繁出现Full GC

**分析**：
1. 使用jstat查看GC情况
   ```
   jstat -gcutil <pid> 1000
   ```
2. 发现老年代使用率持续上升，最终触发Full GC

**解决方案**：
1. 增大堆内存
   ```
   -Xms4g -Xmx4g
   ```
2. 调整年轻代和老年代的比例
   ```
   -XX:NewRatio=2
   ```
3. 检查代码中是否存在内存泄漏

#### 案例2：GC停顿时间过长

**症状**：应用偶尔出现较长时间的停顿

**分析**：
1. 查看GC日志，发现Full GC停顿时间较长

**解决方案**：
1. 使用CMS或G1收集器
   ```
   -XX:+UseG1GC
   -XX:MaxGCPauseMillis=200
   ```
2. 减小堆内存大小，降低单次GC的工作量

## 5. JVM内存模型（JMM）

Java内存模型（Java Memory Model，JMM）是一种抽象的概念，它定义了线程之间如何通过内存进行交互。

### 5.1 主内存与工作内存

JMM规定所有的变量都存储在主内存中，每个线程还有自己的工作内存：

- **主内存**：所有线程共享的内存区域，存储所有变量的主副本
- **工作内存**：每个线程私有，存储该线程使用到的变量的副本

```java
public class JMMDemo {
    private int x = 0;    // 存储在主内存中
    private boolean flag = false;  // 存储在主内存中
    
    public void writer() {
        x = 42;    // 写入工作内存
        flag = true;   // 写入工作内存
    }
    
    public void reader() {
        if (flag) {   // 从工作内存读取
            System.out.println(x);  // 从工作内存读取
        }
    }
}
```

### 5.2 内存间交互操作

JMM定义了8种内存间操作：

1. **lock**：作用于主内存，将变量标识为线程独占
2. **unlock**：作用于主内存，解除独占状态
3. **read**：作用于主内存，读取变量传输到工作内存
4. **load**：作用于工作内存，read后将值放入工作内存副本
5. **use**：作用于工作内存，传输值到执行引擎
6. **assign**：作用于工作内存，执行引擎结果赋值给副本
7. **store**：作用于工作内存，传输值到主内存
8. **write**：作用于主内存，store后写入主内存

### 5.3 volatile关键字

volatile是Java提供的最轻量级的同步机制：

1. **可见性**：对volatile变量的写立即对其他线程可见
2. **有序性**：禁止指令重排序优化

```java
public class VolatileDemo {
    private volatile boolean flag = false;
    
    public void writer() {
        flag = true;  // 修改立即对其他线程可见
    }
    
    public void reader() {
        while (!flag) {
            // 等待flag变为true
        }
        // flag为true时退出循环
    }
}
```

### 5.4 happens-before原则

JMM的happens-before原则保证了多线程操作的可见性：

1. **程序顺序规则**：单线程内，按代码顺序执行
2. **监视器锁规则**：解锁happens-before后续加锁
3. **volatile变量规则**：写happens-before后续读
4. **传递性规则**：A happens-before B，B happens-before C，则A happens-before C

```java
public class HappenBeforeDemo {
    private int x = 0;
    private volatile boolean flag = false;
    
    public void writer() {
        x = 42;            // 1
        flag = true;       // 2
    }
    
    public void reader() {
        if (flag) {        // 3
            System.out.println(x);  // 4
        }
    }
}
```

在上例中，由于volatile的happens-before规则，2 happens-before 3，而1 happens-before 2（程序顺序规则），所以1 happens-before 3（传递性），保证了reader线程能看到writer线程对x的写入。

### 5.5 内存屏障

内存屏障是CPU指令，用于控制特定条件下的重排序和内存可见性。Java中的volatile就是通过插入内存屏障来实现的：

1. **LoadLoad屏障**：确保屏障前的读操作先于屏障后的读操作
2. **StoreStore屏障**：确保屏障前的写操作先于屏障后的写操作
3. **LoadStore屏障**：确保屏障前的读操作先于屏障后的写操作
4. **StoreLoad屏障**：确保屏障前的写操作先于屏障后的读操作

```java
public class MemoryBarrierDemo {
    private volatile int value = 0;
    
    public void write(int v) {
        // StoreStore屏障
        value = v;    // volatile写
        // StoreLoad屏障
    }
    
    public int read() {
        // LoadLoad屏障
        int temp = value;    // volatile读
        // LoadStore屏障
        return temp;
    }
}
```