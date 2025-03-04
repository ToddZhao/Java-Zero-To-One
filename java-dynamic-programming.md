# Day 54: Java数据结构与算法 - 动态规划

## 引言

动态规划（Dynamic Programming，简称DP）是一种通过将复杂问题分解为更简单的子问题来解决问题的算法思想。它适用于有重叠子问题和最优子结构性质的问题，通过存储子问题的解来避免重复计算，从而提高算法效率。本文将深入介绍动态规划的基本概念、解题思路以及在Java中的实现。

## 1. 动态规划基础

### 1.1 什么是动态规划

动态规划是一种将复杂问题分解为简单子问题的方法，它的核心思想是：

1. **分解问题**：将原问题分解为相互依赖的子问题
2. **记忆化**：存储已解决子问题的答案，避免重复计算
3. **自底向上**：从最基本的子问题开始解决，逐步构建更复杂问题的解

### 1.2 动态规划的特征

- **最优子结构**：问题的最优解包含其子问题的最优解
- **重叠子问题**：在求解过程中，相同的子问题会被重复计算
- **状态转移方程**：描述问题状态之间的关系

## 2. 动态规划解题步骤

1. **定义状态**：明确定义子问题
2. **找出状态转移方程**：建立子问题之间的关系
3. **确定边界条件**：确定最基本子问题的解
4. **计算顺序**：通常是自底向上
5. **空间优化**：根据需要优化空间复杂度

## 3. 经典动态规划问题

### 3.1 斐波那契数列

斐波那契数列是动态规划的入门问题，定义为：F(0) = 0, F(1) = 1, F(n) = F(n-1) + F(n-2) (n > 1)。

#### 递归解法（不推荐）

```java
public int fibRecursive(int n) {
    if (n <= 1) {
        return n;
    }
    return fibRecursive(n - 1) + fibRecursive(n - 2);
}
```

时间复杂度：O(2^n)，存在大量重复计算

#### 动态规划解法

```java
public int fibDP(int n) {
    if (n <= 1) {
        return n;
    }
    
    int[] dp = new int[n + 1];
    dp[0] = 0;
    dp[1] = 1;
    
    for (int i = 2; i <= n; i++) {
        dp[i] = dp[i - 1] + dp[i - 2];
    }
    
    return dp[n];
}
```

时间复杂度：O(n)，空间复杂度：O(n)

#### 空间优化解法

```java
public int fibOptimized(int n) {
    if (n <= 1) {
        return n;
    }
    
    int prev = 0;
    int curr = 1;
    
    for (int i = 2; i <= n; i++) {
        int next = prev + curr;
        prev = curr;
        curr = next;
    }
    
    return curr;
}
```

时间复杂度：O(n)，空间复杂度：O(1)

### 3.2 最长递增子序列

给定一个无序的整数数组，找到其中最长上升子序列的长度。

```java
public int lengthOfLIS(int[] nums) {
    if (nums == null || nums.length == 0) {
        return 0;
    }
    
    int n = nums.length;
    int[] dp = new int[n];
    Arrays.fill(dp, 1); // 每个元素自身就是一个长度为1的子序列
    
    int maxLength = 1;
    
    for (int i = 1; i < n; i++) {
        for (int j = 0; j < i; j++) {
            if (nums[i] > nums[j]) {
                dp[i] = Math.max(dp[i], dp[j] + 1);
            }
        }
        maxLength = Math.max(maxLength, dp[i]);
    }
    
    return maxLength;
}
```

时间复杂度：O(n²)，空间复杂度：O(n)

### 3.3 背包问题

#### 0-1背包问题

有N件物品和一个容量为V的背包。第i件物品的重量是w[i]，价值是v[i]。求解将哪些物品装入背包可使价值总和最大。

```java
public int knapsack01(int[] weights, int[] values, int capacity) {
    int n = weights.length;
    int[][] dp = new int[n + 1][capacity + 1];
    
    for (int i = 1; i <= n; i++) {
        for (int j = 1; j <= capacity; j++) {
            if (weights[i - 1] <= j) {
                // 可以放入背包，取放入或不放入的最大值
                dp[i][j] = Math.max(dp[i - 1][j], dp[i - 1][j - weights[i - 1]] + values[i - 1]);
            } else {
                // 不能放入背包
                dp[i][j] = dp[i - 1][j];
            }
        }
    }
    
    return dp[n][capacity];
}
```

时间复杂度：O(n*capacity)，空间复杂度：O(n*capacity)

#### 空间优化的0-1背包

```java
public int knapsack01Optimized(int[] weights, int[] values, int capacity) {
    int n = weights.length;
    int[] dp = new int[capacity + 1];
    
    for (int i = 0; i < n; i++) {
        for (int j = capacity; j >= weights[i]; j--) {
            dp[j] = Math.max(dp[j], dp[j - weights[i]] + values[i]);
        }
    }
    
    return dp[capacity];
}
```

时间复杂度：O(n*capacity)，空间复杂度：O(capacity)

## 4. 实际应用

### 4.1 最短路径问题

```java
public int minPathSum(int[][] grid) {
    if (grid == null || grid.length == 0 || grid[0].length == 0) {
        return 0;
    }
    
    int m = grid.length;
    int n = grid[0].length;
    int[][] dp = new int[m][n];
    
    // 初始化起点
    dp[0][0] = grid[0][0];
    
    // 初始化第一行
    for (int j = 1; j < n; j++) {
        dp[0][j] = dp[0][j - 1] + grid[0][j];
    }
    
    // 初始化第一列
    for (int i = 1; i < m; i++) {
        dp[i][0] = dp[i - 1][0] + grid[i][0];
    }
    
    // 填充dp数组
    for (int i = 1; i < m; i++) {
        for (int j = 1; j < n; j++) {
            dp[i][j] = Math.min(dp[i - 1][j], dp[i][j - 1]) + grid[i][j];
        }
    }
    
    return dp[m - 1][n - 1];
}
```

### 4.2 编辑距离

给定两个单词word1和word2，计算将word1转换成word2所使用的最少操作数。可以进行的操作有：插入、删除、替换。

```java
public int minDistance(String word1, String word2) {
    int m = word1.length();
    int n = word2.length();
    
    // dp[i][j]表示word1的前i个字符转换到word2的前j个字符需要的最少操作数
    int[][] dp = new int[m + 1][n + 1];
    
    // 初始化边界条件
    for (int i = 0; i <= m; i++) {
        dp[i][0] = i; // 删除操作
    }
    
    for (int j = 0; j <= n; j++) {
        dp[0][j] = j; // 插入操作
    }
    
    // 填充dp数组
    for (int i = 1; i <= m; i++) {
        for (int j = 1; j <= n; j++) {
            if (word1.charAt(i - 1) == word2.charAt(j - 1)) {
                dp[i][j] = dp[i - 1][j - 1]; // 字符相同，不需要操作
            } else {
                dp[i][j] = 1 + Math.min(dp[i - 1][j - 1], // 替换
                             Math.min(dp[i - 1][j],     // 删除
                                      dp[i][j - 1]));   // 插入
            }
        }
    }
    
    return dp[m][n];
}
```

## 总结

本文介绍了动态规划的基本概念、解题思路以及常见应用，包括：

1. 动态规划的基本原理和特征
2. 动态规划的解题步骤
3. 经典动态规划问题的解法（斐波那契数列、最长递增子序列、背包问题）
4. 实际应用场景（最短路径、编辑距离）

动态规划是一种强大的算法设计技术，掌握它可以帮助我们解决许多复杂的优化问题。在实际应用中，我们需要根据问题特点，找出状态定义和状态转移方程，并考虑如何优化空间复杂度。

## 参考资源

1. 《算法导论》中的动态规划章节
2. LeetCode动态规划题集
3. Java数据结构与算法分析
4. 动态规划问题解析与实践