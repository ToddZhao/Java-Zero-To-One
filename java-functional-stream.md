# Day 36: Java函数式接口和Stream API

## 引言

Java 8引入的函数式接口和Stream API是Java语言的重要革新，它们使得Java能够支持函数式编程范式，提供了更简洁、更优雅的代码编写方式。本文将详细介绍函数式接口的概念和Stream API的使用方法。

## 1. 函数式接口

### 1.1 概述

函数式接口是只包含一个抽象方法的接口，可以使用lambda表达式来创建该接口的对象。

### 1.2 常用函数式接口

```java
import java.util.function.*;

public class FunctionalInterfaceExample {
    public static void main(String[] args) {
        // Predicate：接收一个参数，返回boolean
        Predicate<String> isEmpty = str -> str.isEmpty();
        System.out.println(isEmpty.test(""));  // true
        
        // Consumer：接收一个参数，无返回值
        Consumer<String> printer = str -> System.out.println(str);
        printer.accept("Hello");  // 输出：Hello
        
        // Function：接收一个参数，返回一个结果
        Function<String, Integer> length = str -> str.length();
        System.out.println(length.apply("Hello"));  // 输出：5
        
        // Supplier：无参数，返回一个结果
        Supplier<String> supplier = () -> "Hello";
        System.out.println(supplier.get());  // 输出：Hello
    }
}
```

## 2. Stream API

### 2.1 概述

Stream API提供了一种声明式的方式来处理数据集合，可以执行过滤、映射、归约等操作。

### 2.2 常用Stream操作

```java
import java.util.Arrays;
import java.util.List;
import java.util.stream.Collectors;

public class StreamExample {
    public static void main(String[] args) {
        List<String> names = Arrays.asList("Alice", "Bob", "Charlie", "David");
        
        // 过滤
        List<String> filteredNames = names.stream()
            .filter(name -> name.startsWith("A"))
            .collect(Collectors.toList());
        
        // 映射
        List<Integer> lengths = names.stream()
            .map(String::length)
            .collect(Collectors.toList());
        
        // 排序
        List<String> sortedNames = names.stream()
            .sorted()
            .collect(Collectors.toList());
        
        // 归约
        int totalLength = names.stream()
            .mapToInt(String::length)
            .sum();
    }
}
```

### 2.3 并行流

```java
import java.util.stream.IntStream;

public class ParallelStreamExample {
    public static void main(String[] args) {
        // 使用并行流计算1到1000000的和
        long start = System.currentTimeMillis();
        int sum = IntStream.rangeClosed(1, 1000000)
            .parallel()
            .sum();
        long end = System.currentTimeMillis();
        System.out.println("Time taken: " + (end - start) + "ms");
    }
}
```

## 3. 实践案例

### 3.1 数据处理示例

```java
import java.util.Arrays;
import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

class Person {
    private String name;
    private int age;
    
    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }
    
    public String getName() { return name; }
    public int getAge() { return age; }
}

public class StreamPracticeExample {
    public static void main(String[] args) {
        List<Person> persons = Arrays.asList(
            new Person("Alice", 25),
            new Person("Bob", 30),
            new Person("Charlie", 35)
        );
        
        // 按年龄分组
        Map<Integer, List<Person>> byAge = persons.stream()
            .collect(Collectors.groupingBy(Person::getAge));
        
        // 获取所有人的平均年龄
        double averageAge = persons.stream()
            .mapToInt(Person::getAge)
            .average()
            .orElse(0.0);
        
        // 获取年龄最大的人
        Person oldest = persons.stream()
            .max((p1, p2) -> p1.getAge() - p2.getAge())
            .orElse(null);
    }
}
```

## 4. 最佳实践

1. 函数式接口使用建议：
   - 优先使用标准函数式接口
   - 保持接口的单一职责
   - 适当使用方法引用

2. Stream操作注意事项：
   - 避免过度使用Stream
   - 注意Stream操作的顺序
   - 合理使用并行流

3. 性能优化：
   - 避免装箱拆箱操作
   - 使用适当的终端操作
   - 考虑数据量大小选择是否使用并行流

## 总结

本文介绍了Java函数式编程的两个重要特性：

1. 函数式接口：提供了支持Lambda表达式的基础
2. Stream API：提供了处理集合的声明式方法

通过合理使用这些特性，我们可以编写出更加简洁、可读性更强的代码，同时在处理大量数据时也能获得更好的性能。

## 参考资源

1. Java官方文档：https://docs.oracle.com/javase/8/docs/api/java/util/stream/package-summary.html
2. Java 8实战
3. 函数式编程最佳实践指南
4. Stream API性能优化指南