# Day 3: Java集合框架 - 功能强大的数据结构和算法工具

## 引言

Java集合框架是Java语言中最重要的API之一，它提供了一系列强大的数据结构和算法工具，帮助开发者高效地组织和管理数据。本文将深入介绍Java集合框架的核心组件和实践应用。

## 1. 集合框架概述

### 1.1 集合框架层次结构

```java
Collection
├── List
│   ├── ArrayList
│   ├── LinkedList
│   └── Vector
├── Set
│   ├── HashSet
│   ├── TreeSet
│   └── LinkedHashSet
└── Queue
    ├── PriorityQueue
    └── Deque

Map
├── HashMap
├── TreeMap
├── LinkedHashMap
└── Hashtable
```

### 1.2 主要接口说明

- Collection：集合层次的根接口
- List：有序集合，允许重复元素
- Set：不允许重复元素的集合
- Queue：队列，通常用于存储待处理元素
- Map：键值对映射关系的集合

## 2. List集合

### 2.1 ArrayList

```java
public class ArrayListDemo {
    public static void main(String[] args) {
        // 创建ArrayList
        List<String> fruits = new ArrayList<>();
        
        // 添加元素
        fruits.add("苹果");
        fruits.add("香蕉");
        fruits.add("橙子");
        
        // 访问元素
        System.out.println("第二个水果：" + fruits.get(1));
        
        // 遍历列表
        for (String fruit : fruits) {
            System.out.println(fruit);
        }
        
        // 使用Lambda表达式遍历
        fruits.forEach(fruit -> System.out.println(fruit));
    }
}
```

### 2.2 LinkedList

```java
public class LinkedListDemo {
    public static void main(String[] args) {
        LinkedList<String> tasks = new LinkedList<>();
        
        // 添加元素到队列
        tasks.offer("任务1");
        tasks.offer("任务2");
        
        // 作为栈使用
        tasks.push("紧急任务");
        
        // 获取并移除第一个元素
        String firstTask = tasks.poll();
        System.out.println("处理任务：" + firstTask);
    }
}
```

## 3. Set集合

### 3.1 HashSet

```java
public class HashSetDemo {
    public static void main(String[] args) {
        Set<String> uniqueNames = new HashSet<>();
        
        // 添加元素
        uniqueNames.add("张三");
        uniqueNames.add("李四");
        uniqueNames.add("张三"); // 重复元素不会被添加
        
        System.out.println("集合大小：" + uniqueNames.size()); // 输出2
        System.out.println("是否包含张三：" + uniqueNames.contains("张三"));
    }
}
```

### 3.2 TreeSet

```java
public class TreeSetDemo {
    public static void main(String[] args) {
        TreeSet<Integer> numbers = new TreeSet<>();
        
        // 添加元素
        numbers.add(5);
        numbers.add(2);
        numbers.add(8);
        
        // 自动排序
        System.out.println("有序数字：" + numbers); // [2, 5, 8]
        
        // 获取子集
        System.out.println("大于3的数字：" + numbers.tailSet(3));
    }
}
```

## 4. Map集合

### 4.1 HashMap

```java
public class HashMapDemo {
    public static void main(String[] args) {
        Map<String, Integer> scores = new HashMap<>();
        
        // 添加键值对
        scores.put("张三", 95);
        scores.put("李四", 88);
        scores.put("王五", 92);
        
        // 获取值
        System.out.println("张三的分数：" + scores.get("张三"));
        
        // 遍历Map
        for (Map.Entry<String, Integer> entry : scores.entrySet()) {
            System.out.println(entry.getKey() + ": " + entry.getValue());
        }
        
        // 使用Lambda表达式遍历
        scores.forEach((name, score) -> 
            System.out.println(name + "的分数是：" + score));
    }
}
```

### 4.2 TreeMap

```java
public class TreeMapDemo {
    public static void main(String[] args) {
        TreeMap<String, Double> prices = new TreeMap<>();
        
        // 添加键值对
        prices.put("香蕉", 5.5);
        prices.put("苹果", 8.0);
        prices.put("橙子", 6.5);
        
        // 按键排序
        System.out.println("按字母顺序排序的水果价格：");
        prices.forEach((fruit, price) -> 
            System.out.println(fruit + ": " + price));
    }
}
```

## 5. 集合工具类

### 5.1 Collections工具类

```java
public class CollectionsDemo {
    public static void main(String[] args) {
        List<Integer> numbers = new ArrayList<>();
        numbers.addAll(Arrays.asList(5, 2, 8, 1, 9));
        
        // 排序
        Collections.sort(numbers);
        System.out.println("排序后：" + numbers);
        
        // 二分查找
        int index = Collections.binarySearch(numbers, 5);
        System.out.println("5的位置：" + index);
        
        // 最大值和最小值
        System.out.println("最大值：" + Collections.max(numbers));
        System.out.println("最小值：" + Collections.min(numbers));
    }
}
```

## 6. 实践案例

### 6.1 购物车实现

```java
public class ShoppingCart {
    private Map<Product, Integer> items = new HashMap<>();
    
    public void addItem(Product product, int quantity) {
        items.merge(product, quantity, Integer::sum);
    }
    
    public void removeItem(Product product) {
        items.remove(product);
    }
    
    public double getTotalPrice() {
        return items.entrySet().stream()
            .mapToDouble(entry -> entry.getKey().getPrice() * entry.getValue())
            .sum();
    }
    
    public void printCart() {
        items.forEach((product, quantity) -> 
            System.out.println(product.getName() + " x " + quantity + 
                " = " + product.getPrice() * quantity));
        System.out.println("总计：" + getTotalPrice());
    }
}
```

### 6.2 成绩管理系统

```java
public class GradeManagement {
    private Map<String, List<Integer>> studentGrades = new HashMap<>();
    
    public void addGrade(String student, int grade) {
        studentGrades.computeIfAbsent(student, k -> new ArrayList<>())
            .add(grade);
    }
    
    public double getAverage(String student) {
        List<Integer> grades = studentGrades.get(student);
        if (grades == null || grades.isEmpty()) {
            return 0.0;
        }
        return grades.stream()
            .mapToInt(Integer::intValue)
            .average()
            .orElse(0.0);
    }
    
    public void printReport() {
        studentGrades.forEach((student, grades) -> {
            System.out.println(student + "的成绩：");
            System.out.println("所有成绩：" + grades);
            System.out.println("平均分：" + getAverage(student));
        });
    }
}
```

## 总结

本文介绍了Java集合框架的核心组件和实践应用，包括：

1. 集合框架的整体架构
2. List、Set、Map等核心接口的使用
3. 各种实现类的特点和适用场景
4. 实用的集合工具类
5. 实际应用案例

通过合理使用Java集合框架，我们可以更高效地组织和处理数据。在实际开发中，要根据具体需求选择合适的集合类型，同时要注意性能和内存消耗。

## 参考资源

1. Java官方文档：https://docs.oracle.com/javase/tutorial/collections/
2. Java集合框架最佳实践指南
3. Effective Java中的集合使用建议
4. Java性能优化实践