# Day 53: Java数据结构与算法 - 查找算法

## 引言

查找算法是计算机科学中最基本和最常用的算法之一，它的目的是在一组数据中找到特定的元素。高效的查找算法对于提高程序性能至关重要。本文将介绍常见的查找算法，分析它们的实现方式、时间复杂度以及适用场景。

## 1. 查找算法基础

### 1.1 什么是查找

查找是在一组数据中寻找特定元素的过程。根据数据是否有序，我们可以选择不同的查找策略。

### 1.2 查找算法的评价指标

- **时间复杂度**：算法执行所需的时间
- **空间复杂度**：算法执行所需的额外空间
- **稳定性**：对于相同的关键字，是否保持原有的相对位置

## 2. 基本查找算法

### 2.1 线性查找（顺序查找）

线性查找是最简单的查找算法，它按顺序遍历数组中的每个元素，直到找到目标元素或遍历完整个数组。

```java
public class LinearSearch {
    public static int search(int[] arr, int target) {
        for (int i = 0; i < arr.length; i++) {
            if (arr[i] == target) {
                return i;
            }
        }
        return -1; // 未找到目标元素
    }
    
    // 使用泛型实现的线性查找
    public static <T> int search(T[] arr, T target) {
        for (int i = 0; i < arr.length; i++) {
            if (arr[i].equals(target)) {
                return i;
            }
        }
        return -1;
    }
}
```

- 时间复杂度：O(n)
- 空间复杂度：O(1)
- 适用场景：数据量较小或数据无序时

### 2.2 二分查找（折半查找）

二分查找要求数据必须是有序的。它通过将查找区间不断折半，每次都与区间中点的值比较，从而快速缩小查找范围。

```java
public class BinarySearch {
    public static int search(int[] arr, int target) {
        int left = 0;
        int right = arr.length - 1;
        
        while (left <= right) {
            int mid = left + (right - left) / 2;
            
            if (arr[mid] == target) {
                return mid;
            }
            
            if (arr[mid] < target) {
                left = mid + 1;
            } else {
                right = mid - 1;
            }
        }
        
        return -1; // 未找到目标元素
    }
    
    // 递归实现的二分查找
    public static int searchRecursive(int[] arr, int target) {
        return binarySearchRecursive(arr, target, 0, arr.length - 1);
    }
    
    private static int binarySearchRecursive(int[] arr, int target, int left, int right) {
        if (left > right) {
            return -1;
        }
        
        int mid = left + (right - left) / 2;
        
        if (arr[mid] == target) {
            return mid;
        }
        
        if (arr[mid] < target) {
            return binarySearchRecursive(arr, target, mid + 1, right);
        }
        
        return binarySearchRecursive(arr, target, left, mid - 1);
    }
}
```

- 时间复杂度：O(log n)
- 空间复杂度：O(1)（迭代版本）或 O(log n)（递归版本）
- 适用场景：数据量大且有序时

## 3. 高级查找算法

### 3.1 插值查找

插值查找是对二分查找的改进，它根据要查找的关键字value与查找区间的关键字分布情况，动态计算出下一次要比较的位置。

```java
public class InterpolationSearch {
    public static int search(int[] arr, int target) {
        int left = 0;
        int right = arr.length - 1;
        
        while (left <= right && target >= arr[left] && target <= arr[right]) {
            if (left == right) {
                if (arr[left] == target) {
                    return left;
                }
                return -1;
            }
            
            // 计算插值位置
            int pos = left + (((right - left) * 
                (target - arr[left])) / 
                (arr[right] - arr[left]));
            
            if (arr[pos] == target) {
                return pos;
            }
            
            if (arr[pos] < target) {
                left = pos + 1;
            } else {
                right = pos - 1;
            }
        }
        
        return -1;
    }
}
```

- 时间复杂度：O(log log n)（最好情况），O(n)（最坏情况）
- 空间复杂度：O(1)
- 适用场景：数据分布比较均匀的有序数组

### 3.2 哈希查找

哈希查找通过哈希函数将关键字映射到数组下标，实现快速查找。

```java
public class HashSearch<K, V> {
    private class Entry<K, V> {
        K key;
        V value;
        Entry<K, V> next;
        
        Entry(K key, V value) {
            this.key = key;
            this.value = value;
        }
    }
    
    private Entry<K, V>[] table;
    private int size;
    private static final int DEFAULT_CAPACITY = 16;
    
    @SuppressWarnings("unchecked")
    public HashSearch() {
        table = new Entry[DEFAULT_CAPACITY];
        size = 0;
    }
    
    public void put(K key, V value) {
        if (key == null) {
            throw new IllegalArgumentException("Key cannot be null");
        }
        
        int index = hash(key);
        Entry<K, V> entry = table[index];
        
        while (entry != null) {
            if (entry.key.equals(key)) {
                entry.value = value;
                return;
            }
            entry = entry.next;
        }
        
        Entry<K, V> newEntry = new Entry<>(key, value);
        newEntry.next = table[index];
        table[index] = newEntry;
        size++;
    }
    
    public V get(K key) {
        if (key == null) {
            throw new IllegalArgumentException("Key cannot be null");
        }
        
        int index = hash(key);
        Entry<K, V> entry = table[index];
        
        while (entry != null) {
            if (entry.key.equals(key)) {
                return entry.value;
            }
            entry = entry.next;
        }
        
        return null;
    }
    
    private int hash(K key) {
        return Math.abs(key.hashCode() % table.length);
    }
}
```

- 时间复杂度：O(1)（平均情况），O(n)（最坏情况）
- 空间复杂度：O(n)
- 适用场景：需要快速查找且内存空间充足时

## 4. 实践应用

### 4.1 数据库索引查找

```java
public class DatabaseIndex {
    private TreeMap<String, List<Integer>> index;
    
    public DatabaseIndex() {
        index = new TreeMap<>();
    }
    
    // 建立索引
    public void buildIndex(String[] keys, int[] values) {
        for (int i = 0; i < keys.length; i++) {
            index.computeIfAbsent(keys[i], k -> new ArrayList<>())
                 .add(values[i]);
        }
    }
    
    // 使用索引查找
    public List<Integer> search(String key) {
        return index.getOrDefault(key, new ArrayList<>());
    }
}
```

## 总结

本文介绍了几种常见的查找算法，包括：

1. 基本查找算法（线性查找、二分查找）
2. 高级查找算法（插值查找、哈希查找）
3. 各种算法的实现方式和性能分析
4. 实际应用场景

在实际开发中，需要根据数据的特点（如数据量、是否有序、内存限制等）选择合适的查找算法。同时，也要考虑算法的时间复杂度和空间复杂度，在效率和资源消耗之间找到平衡。

## 参考资源

1. Java数据结构与算法基础
2. 算法导论中的查找算法章节
3. Java集合框架源码
4. 数据库索引设计与优化