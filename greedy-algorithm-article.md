# Java数据结构与算法 - 贪心算法详解

## 什么是贪心算法？

贪心算法（Greedy Algorithm）是一种在每一步选择中都采取当前状态下最好或最优的选择，从而希望导致结果是最好或最优的算法。贪心算法不从整体最优考虑，它所作出的选择只是在某种意义上的局部最优选择。

贪心算法的核心思想可以概括为：
1. 建立数学模型来描述问题
2. 把求解的问题分成若干个子问题
3. 对每个子问题求解，得到子问题的局部最优解
4. 把子问题的解合成原来解问题的一个解

## 贪心算法的基本要素

贪心算法具备两个重要的基本要素：

1. **贪心选择性质**：一个问题的整体最优解可以通过一系列局部最优的选择来达到。每次做出的贪心选择只依赖于当前状态，而不依赖于以后的选择。

2. **最优子结构**：问题的最优解包含其子问题的最优解。利用这种结构，贪心算法通过做出一系列最优选择来构造出问题的最优解。

## 经典应用场景

让我们通过几个经典的问题来深入理解贪心算法：

### 1. 找零钱问题

假设我们有面值为1元、5元、10元、20元、50元、100元的钞票，需要找给顾客41元零钱，如何用最少的钞票组合？

```java
public class ChangeMaking {
    public static void makeChange(int amount) {
        int[] denominations = {100, 50, 20, 10, 5, 1};  // 从大到小排序的钞票面值
        
        System.out.println("需要找零: " + amount + "元");
        
        for (int denomination : denominations) {
            if (amount >= denomination) {
                int count = amount / denomination;  // 当前面值需要的数量
                amount = amount % denomination;     // 更新剩余金额
                System.out.println("需要" + denomination + "元" + count + "张");
            }
        }
    }

    public static void main(String[] args) {
        makeChange(41);
    }
}
```

在这个例子中，我们采用了"优先使用面值最大的钞票"的贪心策略。对于41元，算法会输出：
- 需要20元2张
- 需要1元1张

### 2. 活动选择问题

假设有n个活动，每个活动都有开始时间和结束时间，要在同一个会议室安排尽可能多的活动。

```java
public class ActivitySelection {
    static class Activity implements Comparable<Activity> {
        int start, finish;
        
        public Activity(int start, int finish) {
            this.start = start;
            this.finish = finish;
        }
        
        @Override
        public int compareTo(Activity other) {
            return this.finish - other.finish;
        }
    }
    
    public static void selectActivities(Activity[] activities) {
        // 按结束时间排序
        Arrays.sort(activities);
        
        System.out.println("选择的活动为：");
        
        // 选择第一个活动
        Activity lastSelected = activities[0];
        System.out.printf("活动：开始时间=%d，结束时间=%d\n", 
            lastSelected.start, lastSelected.finish);
        
        // 遍历剩余活动
        for (int i = 1; i < activities.length; i++) {
            if (activities[i].start >= lastSelected.finish) {
                System.out.printf("活动：开始时间=%d，结束时间=%d\n",
                    activities[i].start, activities[i].finish);
                lastSelected = activities[i];
            }
        }
    }
    
    public static void main(String[] args) {
        Activity[] activities = {
            new Activity(1, 4),
            new Activity(3, 5),
            new Activity(0, 6),
            new Activity(5, 7),
            new Activity(3, 9),
            new Activity(5, 9),
            new Activity(6, 10),
            new Activity(8, 11),
            new Activity(8, 12),
            new Activity(2, 14),
        };
        
        selectActivities(activities);
    }
}
```

在这个问题中，我们采用了"优先选择结束时间最早的活动"的贪心策略。

### 3. Huffman编码

Huffman编码是一种经典的数据压缩算法，也是贪心算法的典型应用：

```java
public class HuffmanCoding {
    static class Node implements Comparable<Node> {
        char ch;
        int freq;
        Node left, right;
        
        Node(char ch, int freq) {
            this.ch = ch;
            this.freq = freq;
        }
        
        Node(Node left, Node right) {
            this.freq = left.freq + right.freq;
            this.left = left;
            this.right = right;
        }
        
        @Override
        public int compareTo(Node other) {
            return this.freq - other.freq;
        }
    }
    
    public static void printHuffmanCodes(Node root, String code) {
        if (root == null) return;
        
        // 如果是叶子节点，打印编码
        if (root.left == null && root.right == null) {
            System.out.println("字符：" + root.ch + " 编码：" + code);
            return;
        }
        
        // 递归处理左右子树
        printHuffmanCodes(root.left, code + "0");
        printHuffmanCodes(root.right, code + "1");
    }
    
    public static void buildHuffmanTree(char[] chars, int[] freq) {
        PriorityQueue<Node> pq = new PriorityQueue<>();
        
        // 创建初始节点
        for (int i = 0; i < chars.length; i++) {
            pq.offer(new Node(chars[i], freq[i]));
        }
        
        // 构建Huffman树
        while (pq.size() > 1) {
            Node left = pq.poll();
            Node right = pq.poll();
            pq.offer(new Node(left, right));
        }
        
        // 打印Huffman编码
        printHuffmanCodes(pq.peek(), "");
    }
    
    public static void main(String[] args) {
        char[] chars = {'a', 'b', 'c', 'd', 'e'};
        int[] freq = {4, 7, 1, 3, 2};
        
        buildHuffmanTree(chars, freq);
    }
}
```

在Huffman编码中，我们采用了"优先合并频率最小的两个节点"的贪心策略。

## 贪心算法的优缺点

优点：
- 简单直观，容易实现
- 执行效率高
- 在某些问题上能得到最优解

缺点：
- 不能保证得到全局最优解
- 可能得到次优解
- 不是所有问题都适合使用贪心算法

## 如何判断问题是否适合使用贪心算法？

1. 问题能够分解成子问题
2. 子问题的最优解能递推到最终问题的最优解
3. 贪心选择性质：每一步的最优解都包含上一步的最优解

## 总结

贪心算法是一种在某些特定问题上非常有效的算法策略。虽然它不能保证在所有情况下都能得到最优解，但在满足贪心选择性质和最优子结构性质的问题上，它能够快速得到一个可接受的解，而且实现起来相对简单。在实际应用中，我们需要根据具体问题的特点来判断是否适合使用贪心算法。
