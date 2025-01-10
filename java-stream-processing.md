# Day 68: Java大数据 - 数据流处理

## 1. 引言

在大数据处理中，数据流(Stream)是一个非常重要的概念。Java 8引入的Stream API为我们提供了强大的数据处理能力，让我们能够以函数式编程的方式处理集合数据。本文将深入探讨Java中的数据流处理，并通过实际案例来展示其应用。

## 2. 什么是Stream？

Stream（流）是Java 8引入的全新概念，它代表了数据源的序列。与集合不同，Stream不会存储数据，而是像流水一样，通过管道的方式对数据进行操作。Stream的特点包括：

- 不存储数据：Stream是对数据源的映射，不改变原数据
- 函数式编程：采用函数式编程方式处理数据
- 惰性执行：中间操作不会立即执行
- 可并行：支持并行操作，提高处理效率

## 3. Stream操作的基本步骤

Stream操作主要包含三个步骤：

1. 创建Stream
2. 进行中间操作（中间操作可以有多个）
3. 执行终止操作（只能有一个）

让我们通过代码来详细说明这些步骤：

```java
import java.util.Arrays;
import java.util.List;
import java.util.stream.Collectors;

public class StreamDemo {
    public static void main(String[] args) {
        // 1. 创建Stream
        List<String> names = Arrays.asList("Alice", "Bob", "Charlie", "David", "Eva");
        
        // 2. 中间操作 + 3. 终止操作
        List<String> filteredNames = names.stream()    // 创建流
            .filter(name -> name.length() > 3)         // 中间操作：过滤
            .map(String::toUpperCase)                  // 中间操作：转换
            .sorted()                                  // 中间操作：排序
            .collect(Collectors.toList());             // 终止操作：收集结果
            
        System.out.println("处理后的名字：" + filteredNames);
    }
}
```

## 4. Stream的创建方式

Stream可以通过多种方式创建，下面展示几种常见的创建方式：

```java
import java.util.Arrays;
import java.util.stream.Stream;

public class StreamCreationDemo {
    public static void main(String[] args) {
        // 1. 从集合创建
        List<Integer> list = Arrays.asList(1, 2, 3, 4, 5);
        Stream<Integer> streamFromList = list.stream();

        // 2. 从数组创建
        String[] arr = {"a", "b", "c"};
        Stream<String> streamFromArray = Arrays.stream(arr);

        // 3. 使用Stream.of()
        Stream<Integer> streamFromValues = Stream.of(1, 2, 3, 4, 5);

        // 4. 创建无限流
        Stream<Integer> infiniteStream = Stream.iterate(0, n -> n + 2);
        // 注意：无限流需要限制才能使用
        infiniteStream.limit(5).forEach(System.out::println);
    }
}
```

## 5. 常用的Stream操作

### 5.1 中间操作

让我们通过一个完整的示例来展示常用的中间操作：

```java
import java.util.Arrays;
import java.util.List;

public class StreamOperationsDemo {
    public static void main(String[] args) {
        List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);

        numbers.stream()
            .filter(n -> n % 2 == 0)     // 过滤偶数
            .map(n -> n * n)             // 平方
            .peek(n -> System.out.println("处理中的数据：" + n))  // 查看中间结果
            .limit(3)                    // 限制结果数量
            .forEach(System.out::println);
    }
}
```

### 5.2 终止操作

下面是一些常用终止操作的示例：

```java
import java.util.List;
import java.util.Optional;
import java.util.stream.Collectors;

public class StreamTerminalOperationsDemo {
    public static void main(String[] args) {
        List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5);

        // 1. 收集为List
        List<Integer> collectedList = numbers.stream()
            .filter(n -> n % 2 == 0)
            .collect(Collectors.toList());

        // 2. 查找第一个元素
        Optional<Integer> firstEven = numbers.stream()
            .filter(n -> n % 2 == 0)
            .findFirst();

        // 3. 归约操作
        int sum = numbers.stream()
            .reduce(0, Integer::sum);

        // 4. 统计信息
        long count = numbers.stream()
            .filter(n -> n > 3)
            .count();

        System.out.println("偶数列表：" + collectedList);
        System.out.println("第一个偶数：" + firstEven.orElse(null));
        System.out.println("总和：" + sum);
        System.out.println("大于3的数量：" + count);
    }
}
```

## 6. 实际应用示例：员工数据分析

让我们通过一个更实际的例子来展示Stream的强大功能：

```java
import java.util.Arrays;
import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

class Employee {
    private String name;
    private String department;
    private double salary;

    public Employee(String name, String department, double salary) {
        this.name = name;
        this.department = department;
        this.salary = salary;
    }

    // Getters
    public String getName() { return name; }
    public String getDepartment() { return department; }
    public double getSalary() { return salary; }
}

public class EmployeeAnalysisDemo {
    public static void main(String[] args) {
        List<Employee> employees = Arrays.asList(
            new Employee("张三", "技术部", 10000),
            new Employee("李四", "市场部", 8000),
            new Employee("王五", "技术部", 12000),
            new Employee("赵六", "市场部", 9000)
        );

        // 1. 按部门对员工分组
        Map<String, List<Employee>> byDepartment = employees.stream()
            .collect(Collectors.groupingBy(Employee::getDepartment));

        // 2. 计算每个部门的平均工资
        Map<String, Double> avgSalaryByDept = employees.stream()
            .collect(Collectors.groupingBy(
                Employee::getDepartment,
                Collectors.averagingDouble(Employee::getSalary)
            ));

        // 3. 找出工资最高的员工
        Optional<Employee> highestPaid = employees.stream()
            .max(Comparator.comparingDouble(Employee::getSalary));

        // 输出结果
        System.out.println("===== 部门员工列表 =====");
        byDepartment.forEach((dept, emps) -> {
            System.out.println(dept + ":");
            emps.forEach(emp -> System.out.println("  - " + emp.getName()));
        });

        System.out.println("\n===== 部门平均工资 =====");
        avgSalaryByDept.forEach((dept, avg) -> 
            System.out.printf("%s: %.2f\n", dept, avg));

        System.out.println("\n===== 最高工资员工 =====");
        highestPaid.ifPresent(emp -> 
            System.out.printf("%s (工资: %.2f)\n", emp.getName(), emp.getSalary()));
    }
}
```

## 7. 性能考虑和最佳实践

在使用Stream API时，需要注意以下几点：

1. **合理使用并行流**：
   - 数据量大时考虑使用并行流
   - 注意并行流的开销
   - 确保操作是线程安全的

```java
// 并行流示例
public class ParallelStreamDemo {
    public static void main(String[] args) {
        List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);

        // 使用并行流计算平方和
        long start = System.currentTimeMillis();
        int sum = numbers.parallelStream()
            .map(n -> {
                try {
                    Thread.sleep(100); // 模拟耗时操作
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                return n * n;
            })
            .reduce(0, Integer::sum);

        System.out.println("计算结果：" + sum);
        System.out.println("耗时：" + (System.currentTimeMillis() - start) + "ms");
    }
}
```

2. **避免过度使用Stream**：
   - 简单操作使用传统循环
   - 确保代码的可读性
   - 注意Stream操作的顺序

## 8. 总结

Java Stream API提供了一种优雅且强大的方式来处理数据集合。通过本文的学习，我们了解了：

- Stream的基本概念和特点
- Stream的创建方式
- 常用的中间操作和终止操作
- 实际应用场景
- 性能优化建议

掌握Stream API不仅能提高代码的可读性，还能帮助我们更好地处理大数据集合。在实际开发中，要根据具体场景选择合适的方式来处理数据。
