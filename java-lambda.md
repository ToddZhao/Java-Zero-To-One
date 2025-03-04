# Day 8: Java Lambda表达式和函数式编程

## 引言

Java 8引入的Lambda表达式是一个重要的特性，它为Java添加了函数式编程的能力。Lambda表达式使代码更简洁、更具表达力，并且能够更好地支持并行处理。本文将详细介绍Lambda表达式和函数式编程的核心概念及其实践应用。

## 1. Lambda表达式基础

### 1.1 什么是Lambda表达式

Lambda表达式是一种匿名函数，它可以作为参数传递给方法：

```java
// 传统方式
Button button = new Button();
button.addActionListener(new ActionListener() {
    @Override
    public void actionPerformed(ActionEvent e) {
        System.out.println("Button clicked");
    }
});

// Lambda表达式
button.addActionListener(e -> System.out.println("Button clicked"));
```

### 1.2 Lambda表达式语法

```java
// 无参数
() -> System.out.println("Hello World")

// 单个参数
x -> x * x

// 多个参数
(x, y) -> x + y

// 代码块
(x, y) -> {
    int sum = x + y;
    return sum;
}
```

## 2. 函数式接口

### 2.1 常用函数式接口

```java
// Predicate：接收一个参数，返回boolean
Predicate<String> isEmpty = str -> str.isEmpty();

// Consumer：接收一个参数，无返回值
Consumer<String> printer = str -> System.out.println(str);

// Function：接收一个参数，返回一个结果
Function<String, Integer> length = str -> str.length();

// Supplier：无参数，返回一个结果
Supplier<String> getter = () -> "Hello World";
```

### 2.2 自定义函数式接口

```java
@FunctionalInterface
public interface Calculator {
    int calculate(int x, int y);
    
    // 可以包含默认方法
    default void printInfo() {
        System.out.println("这是一个计算器接口");
    }
}

// 使用自定义函数式接口
Calculator add = (x, y) -> x + y;
Calculator multiply = (x, y) -> x * y;

System.out.println(add.calculate(5, 3));      // 输出：8
System.out.println(multiply.calculate(5, 3)); // 输出：15
```

## 3. 方法引用

### 3.1 方法引用类型

```java
// 静态方法引用
Function<String, Integer> parseInt = Integer::parseInt;

// 实例方法引用
String str = "Hello";
Supplier<Integer> length = str::length;

// 构造方法引用
Supplier<ArrayList<String>> listCreator = ArrayList::new;

// 对象方法引用
List<String> list = Arrays.asList("apple", "banana", "orange");
list.forEach(System.out::println);
```

## 4. Stream API

### 4.1 创建Stream

```java
// 从集合创建
List<String> list = Arrays.asList("a", "b", "c");
Stream<String> stream1 = list.stream();

// 从数组创建
String[] array = {"a", "b", "c"};
Stream<String> stream2 = Arrays.stream(array);

// 使用Stream.of
Stream<String> stream3 = Stream.of("a", "b", "c");
```

### 4.2 Stream操作

```java
public class StreamExample {
    public static void main(String[] args) {
        List<String> names = Arrays.asList("Alice", "Bob", "Charlie", "David");
        
        // 过滤和转换
        List<String> filteredNames = names.stream()
            .filter(name -> name.length() > 4)
            .map(String::toUpperCase)
            .collect(Collectors.toList());
        
        // 排序
        List<String> sortedNames = names.stream()
            .sorted()
            .collect(Collectors.toList());
        
        // 统计
        long count = names.stream()
            .filter(name -> name.startsWith("A"))
            .count();
    }
}
```

## 5. 实践案例

### 5.1 集合处理

```java
public class CollectionProcessor {
    public static void processEmployees(List<Employee> employees) {
        // 获取高薪员工
        List<Employee> highPaidEmployees = employees.stream()
            .filter(e -> e.getSalary() > 10000)
            .sorted(Comparator.comparing(Employee::getSalary).reversed())
            .collect(Collectors.toList());
        
        // 计算平均工资
        double averageSalary = employees.stream()
            .mapToDouble(Employee::getSalary)
            .average()
            .orElse(0.0);
        
        // 按部门分组
        Map<String, List<Employee>> byDepartment = employees.stream()
            .collect(Collectors.groupingBy(Employee::getDepartment));
    }
}
```

### 5.2 并行处理

```java
public class ParallelProcessor {
    public static long countWords(List<String> documents) {
        return documents.parallelStream()
            .mapToLong(doc -> doc.split("\\s+").length)
            .sum();
    }
    
    public static List<Integer> processNumbers(List<Integer> numbers) {
        return numbers.parallelStream()
            .filter(n -> n % 2 == 0)
            .map(n -> n * n)
            .sorted()
            .collect(Collectors.toList());
    }
}
```

## 6. 最佳实践

1. 优先使用标准函数式接口
2. 保持Lambda表达式简短清晰
3. 避免在Lambda表达式中使用复杂的逻辑
4. 合理使用方法引用
5. 注意并行流的使用场景

## 总结

本文介绍了Java Lambda表达式和函数式编程的核心概念及实践应用，包括：

1. Lambda表达式的基本语法
2. 函数式接口的使用
3. 方法引用的类型
4. Stream API的操作
5. 实践案例和最佳实践

通过掌握这些知识，我们可以编写更简洁、更具表达力的代码，提高开发效率和代码质量。

## 参考资源

1. Java官方文档：https://docs.oracle.com/javase/tutorial/java/javaOO/lambdaexpressions.html
2. Java 8 Stream API指南
3. 函数式编程最佳实践
4. Lambda表达式性能优化建议