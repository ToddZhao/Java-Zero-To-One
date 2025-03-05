# Day 107: Java JVM进阶 - GC调优实战

## 1. GC调优基本原则

### 1.1 调优目标

- 降低Full GC频率
- 减少GC停顿时间
- 提高吞吐量
- 避免内存泄漏
- 合理利用系统资源

### 1.2 调优步骤

1. 确定目标
2. 收集数据
3. 分析问题
4. 调整参数
5. 验证效果

## 2. GC调优工具

### 2.1 命令行工具

```bash
# 查看JVM参数
jinfo -flags <pid>

# 查看GC统计信息
jstat -gcutil <pid> 1000

# 查看堆内存使用情况
jmap -heap <pid>

# 导出堆转储文件
jmap -dump:format=b,file=heap.bin <pid>
```

### 2.2 可视化工具

#### JConsole

```java
public class JConsoleDemo {
    private static List<byte[]> list = new ArrayList<>();
    
    public static void main(String[] args) throws InterruptedException {
        // 每秒创建1MB对象
        while (true) {
            list.add(new byte[1024 * 1024]);
            Thread.sleep(1000);
        }
    }
}
```

#### VisualVM

- CPU分析
- 内存分析
- 线程分析
- Visual GC插件

## 3. 常见GC问题分析

### 3.1 频繁Full GC

```java
public class FrequentFullGCDemo {
    private static Map<String, byte[]> cache = new HashMap<>();
    
    public static void main(String[] args) throws InterruptedException {
        int count = 0;
        while (true) {
            cache.put("key" + count, new byte[1024 * 1024]); // 1MB
            count++;
            Thread.sleep(100);
            
            if (count % 100 == 0) {
                System.out.println("当前缓存大小: " + cache.size());
            }
        }
    }
}
```

**问题分析**：
1. 大对象直接进入老年代
2. 内存泄漏导致老年代填满
3. 年轻代存活对象太多

**解决方案**：
1. 增大年轻代空间
   ```
   -Xmn4g
   ```
2. 调整大对象阈值
   ```
   -XX:PretenureSizeThreshold=3m
   ```
3. 检查内存泄漏

### 3.2 GC停顿时间过长

```java
public class LongPauseDemo {
    private static List<byte[]> list = new ArrayList<>();
    
    public static void main(String[] args) throws InterruptedException {
        while (true) {
            for (int i = 0; i < 100; i++) {
                list.add(new byte[1024 * 1024]); // 1MB
            }
            Thread.sleep(100);
            
            // 模拟清理
            if (list.size() > 1000) {
                list.subList(0, 500).clear();
                System.out.println("清理完成，当前大小: " + list.size());
            }
        }
    }
}
```

**问题分析**：
1. 单次GC工作量大
2. 老年代空间碎片化
3. 垃圾收集器选择不当

**解决方案**：
1. 使用G1收集器
   ```
   -XX:+UseG1GC
   -XX:MaxGCPauseMillis=200
   ```
2. 控制堆内存大小
3. 及时清理垃圾对象

## 4. 实战案例分析

### 4.1 案例一：Web应用响应时间优化

**现象**：
- 应用偶发性响应慢
- CPU使用率正常
- 内存使用率波动大

**分析过程**：
1. 使用jstat观察GC情况
   ```bash
   jstat -gcutil <pid> 1000
   ```
2. 发现Full GC频繁发生
3. 导出堆转储文件分析
   ```bash
   jmap -dump:format=b,file=heap.bin <pid>
   ```

**解决方案**：
1. 增大年轻代空间
   ```
   -Xmn4g
   ```
2. 启用并发标记清除收集器
   ```
   -XX:+UseConcMarkSweepGC
   ```
3. 优化缓存策略

### 4.2 案例二：批处理应用内存泄漏

```java
public class BatchProcessDemo {
    private static Map<String, SoftReference<byte[]>> cache = new HashMap<>();
    
    public static void main(String[] args) throws InterruptedException {
        while (true) {
            // 模拟批处理任务
            processBatch();
            Thread.sleep(1000);
        }
    }
    
    private static void processBatch() {
        for (int i = 0; i < 100; i++) {
            String key = "data" + i;
            cache.put(key, new SoftReference<>(new byte[1024 * 1024]));
        }
        
        // 处理缓存数据
        for (Map.Entry<String, SoftReference<byte[]>> entry : cache.entrySet()) {
            byte[] data = entry.getValue().get();
            if (data != null) {
                // 处理数据
            }
        }
    }
}
```

**问题分析**：
1. 使用jmap分析堆内存
2. 发现SoftReference对象堆积
3. 缓存清理不及时

**解决方案**：
1. 使用WeakHashMap替代HashMap
2. 设置合理的缓存过期策略
3. 定期清理无效引用

## 5. GC调优最佳实践

### 5.1 调优建议

1. **合理设置内存大小**
   ```
   -Xms4g -Xmx4g -Xmn2g
   ```

2. **选择合适的垃圾收集器**
   - 响应时间优先：CMS、G1
   - 吞吐量优先：Parallel GC

3. **避免创建大量临时对象**
   ```java
   // 不推荐
   for (int i = 0; i < 1000; i++) {
       String result = "value" + i;
   }
   
   // 推荐
   StringBuilder sb = new StringBuilder();
   for (int i = 0; i < 1000; i++) {
       sb.append("value").append(i);
   }
   ```

4. **及时释放不用的对象引用**
   ```java
   public class ResourceManager {
       private byte[] data;
       
       public void process() {
           data = new byte[1024 * 1024];
           // 处理数据
           data = null; // 显式释放引用
       }
   }
   ```

### 5.2 监控方案

1. **JVM参数配置**
   ```
   -XX:+PrintGCDetails
   -XX:+PrintGCTimeStamps
   -Xloggc:gc.log
   ```

2. **监控指标**
   - GC频率和时间
   - 内存使用率
   - 线程状态
   - CPU使用率

3. **告警策略**
   - Full GC频率超阈值
   - GC停顿时间过长
   - 内存使用率持续高位

### 5.3 调优工具链

1. **问题发现**
   - jstat：实时监控
   - jstack：线程分析
   - jmap：内存分析

2. **问题分析**
   - MAT：堆转储分析
   - JProfiler：性能分析
   - GCViewer：GC日志分析

3. **问题解决**
   - JMC：实时诊断
   - Arthas：在线排查
   - BTrace：动态追踪

通过本文的学习，你应该能够：
1. 理解GC调优的基本原则和方法
2. 掌握常用的GC调优工具
3. 分析和解决常见的GC问题
4. 制定合理的GC调优方案
5. 建立完整的GC监控体系

记住，GC调优是一个持续的过程，需要根据应用的实际情况不断调整和优化。同时，过早优化是万恶之源，在进行GC调优之前，应该先确认是否真的存在性能问题。