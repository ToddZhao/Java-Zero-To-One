# Day 52: Java数据结构与算法 - 排序算法

## 引言

排序是计算机科学中最基本也是最重要的算法之一，它将一组数据按照特定的顺序重新排列。排序算法在数据处理、搜索优化和数据分析等领域有着广泛的应用。本文将深入介绍常见的排序算法，分析它们的时间复杂度、空间复杂度以及适用场景。

## 1. 排序算法基础

### 1.1 什么是排序

排序是将一组数据按照特定的顺序（如升序或降序）重新排列的过程。

### 1.2 排序算法的评价指标

- **时间复杂度**：算法执行所需的时间
- **空间复杂度**：算法执行所需的额外空间
- **稳定性**：相同元素在排序前后的相对位置是否发生变化
- **原地排序**：是否需要额外的存储空间

## 2. 基本排序算法

### 2.1 冒泡排序

冒泡排序是最简单的排序算法之一，它通过重复地遍历要排序的数列，比较相邻的两个元素，如果它们的顺序错误就交换它们，直到没有再需要交换的元素为止。

```java
public void bubbleSort(int[] arr) {
    int n = arr.length;
    boolean swapped;
    
    for (int i = 0; i < n - 1; i++) {
        swapped = false;
        
        for (int j = 0; j < n - i - 1; j++) {
            if (arr[j] > arr[j + 1]) {
                // 交换 arr[j] 和 arr[j+1]
                int temp = arr[j];
                arr[j] = arr[j + 1];
                arr[j + 1] = temp;
                swapped = true;
            }
        }
        
        // 如果内层循环未发生交换，则数组已排序
        if (!swapped) {
            break;
        }
    }
}
```

- 时间复杂度：O(n²)
- 空间复杂度：O(1)
- 稳定性：稳定

### 2.2 选择排序

选择排序是一种简单直观的排序算法，它的工作原理是每次从未排序区间中找到最小（或最大）元素，放到已排序区间的末尾。

```java
public void selectionSort(int[] arr) {
    int n = arr.length;
    
    for (int i = 0; i < n - 1; i++) {
        int minIndex = i;
        
        for (int j = i + 1; j < n; j++) {
            if (arr[j] < arr[minIndex]) {
                minIndex = j;
            }
        }
        
        // 交换 arr[i] 和 arr[minIndex]
        int temp = arr[i];
        arr[i] = arr[minIndex];
        arr[minIndex] = temp;
    }
}
```

- 时间复杂度：O(n²)
- 空间复杂度：O(1)
- 稳定性：不稳定

### 2.3 插入排序

插入排序是一种简单直观的排序算法，它的工作原理是通过构建有序序列，对于未排序数据，在已排序序列中从后向前扫描，找到相应位置并插入。

```java
public void insertionSort(int[] arr) {
    int n = arr.length;
    
    for (int i = 1; i < n; i++) {
        int key = arr[i];
        int j = i - 1;
        
        // 将比key大的元素向后移动
        while (j >= 0 && arr[j] > key) {
            arr[j + 1] = arr[j];
            j--;
        }
        
        arr[j + 1] = key;
    }
}
```

- 时间复杂度：O(n²)
- 空间复杂度：O(1)
- 稳定性：稳定

## 3. 高级排序算法

### 3.1 快速排序

快速排序是一种分治算法，它通过选择一个"基准