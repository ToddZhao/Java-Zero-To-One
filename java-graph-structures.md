# Day 51: Java数据结构与算法 - 图

## 引言

图是一种非常重要的非线性数据结构，它由顶点（节点）和边组成，用于表示元素之间的关系。图在现实生活中有着广泛的应用，如社交网络、地图导航、网络拓扑等。本文将深入介绍图的概念、实现方式以及常见算法。

## 1. 图的基础概念

### 1.1 什么是图

图是由顶点集合和边集合组成的数据结构，可以表示为 G = (V, E)，其中：

- V 是顶点（Vertex）的集合
- E 是边（Edge）的集合

### 1.2 图的基本术语

- **顶点（Vertex）**：图中的基本单位，也称为节点
- **边（Edge）**：连接两个顶点的线段
- **权重（Weight）**：边上的数值，表示两个顶点之间的关系强度
- **度（Degree）**：与顶点相连的边的数量
- **路径（Path）**：从一个顶点到另一个顶点经过的边的序列
- **环（Cycle）**：起点和终点相同的路径

### 1.3 图的分类

1. 按照边的方向：
   - 有向图：边有方向
   - 无向图：边无方向

2. 按照权重：
   - 带权图：边有权重
   - 无权图：边无权重

## 2. 图的表示方法

### 2.1 邻接矩阵

```java
public class AdjacencyMatrix {
    private int[][] matrix;
    private int vertices;
    
    public AdjacencyMatrix(int vertices) {
        this.vertices = vertices;
        matrix = new int[vertices][vertices];
    }
    
    // 添加边
    public void addEdge(int source, int destination) {
        // 无向图需要双向添加
        matrix[source][destination] = 1;
        matrix[destination][source] = 1;
    }
    
    // 删除边
    public void removeEdge(int source, int destination) {
        matrix[source][destination] = 0;
        matrix[destination][source] = 0;
    }
    
    // 打印图
    public void printGraph() {
        for (int i = 0; i < vertices; i++) {
            for (int j = 0; j < vertices; j++) {
                System.out.print(matrix[i][j] + " ");
            }
            System.out.println();
        }
    }
}
```

### 2.2 邻接表

```java
public class AdjacencyList {
    private ArrayList<ArrayList<Integer>> adjList;
    private int vertices;
    
    public AdjacencyList(int vertices) {
        this.vertices = vertices;
        adjList = new ArrayList<>(vertices);
        for (int i = 0; i < vertices; i++) {
            adjList.add(new ArrayList<>());
        }
    }
    
    // 添加边
    public void addEdge(int source, int destination) {
        // 无向图需要双向添加
        adjList.get(source).add(destination);
        adjList.get(destination).add(source);
    }
    
    // 删除边
    public void removeEdge(int source, int destination) {
        adjList.get(source).remove(Integer.valueOf(destination));
        adjList.get(destination).remove(Integer.valueOf(source));
    }
    
    // 打印图
    public void printGraph() {
        for (int i = 0; i < vertices; i++) {
            System.out.print("顶点 " + i + " 的邻接点: ");
            for (Integer vertex : adjList.get(i)) {
                System.out.print(vertex + " ");
            }
            System.out.println();
        }
    }
}
```

## 3. 图的遍历算法

### 3.1 广度优先搜索（BFS）

```java
public void bfs(int startVertex) {
    boolean[] visited = new boolean[vertices];
    Queue<Integer> queue = new LinkedList<>();
    
    visited[startVertex] = true;
    queue.offer(startVertex);
    
    while (!queue.isEmpty()) {
        int vertex = queue.poll();
        System.out.print(vertex + " ");
        
        for (Integer adjacent : adjList.get(vertex)) {
            if (!visited[adjacent]) {
                visited[adjacent] = true;
                queue.offer(adjacent);
            }
        }
    }
}
```

### 3.2 深度优先搜索（DFS）

```java
public void dfs(int startVertex) {
    boolean[] visited = new boolean[vertices];
    dfsUtil(startVertex, visited);
}

private void dfsUtil(int vertex, boolean[] visited) {
    visited[vertex] = true;
    System.out.print(vertex + " ");
    
    for (Integer adjacent : adjList.get(vertex)) {
        if (!visited[adjacent]) {
            dfsUtil(adjacent, visited);
        }
    }
}
```

## 4. 最短路径算法

### 4.1 Dijkstra算法

```java
public void dijkstra(int[][] graph, int source) {
    int vertices = graph.length;
    int[] distance = new int[vertices];
    boolean[] visited = new boolean[vertices];
    
    // 初始化距离数组
    Arrays.fill(distance, Integer.MAX_VALUE);
    distance[source] = 0;
    
    for (int i = 0; i < vertices - 1; i++) {
        int minVertex = findMinDistance(distance, visited);
        visited[minVertex] = true;
        
        for (int j = 0; j < vertices; j++) {
            if (!visited[j] && graph[minVertex][j] != 0 && 
                distance[minVertex] != Integer.MAX_VALUE && 
                distance[minVertex] + graph[minVertex][j] < distance[j]) {
                distance[j] = distance[minVertex] + graph[minVertex][j];
            }
        }
    }
    
    printSolution(distance);
}
```

## 5. 实践应用

### 5.1 社交网络分析

```java
public class SocialNetwork {
    private AdjacencyList network;
    
    public SocialNetwork(int users) {
        network = new AdjacencyList(users);
    }
    
    // 添加好友关系
    public void addFriendship(int user1, int user2) {
        network.addEdge(user1, user2);
    }
    
    // 获取用户的好友列表
    public List<Integer> getFriends(int user) {
        return network.adjList.get(user);
    }
    
    // 计算两个用户之间的社交距离
    public int socialDistance(int user1, int user2) {
        // 使用BFS计算最短路径
        return calculateShortestPath(user1, user2);
    }
}
```

## 总结

本文介绍了图这一重要的数据结构，包括：

1. 图的基本概念和术语
2. 图的表示方法（邻接矩阵和邻接表）
3. 图的遍历算法（BFS和DFS）
4. 最短路径算法（Dijkstra）
5. 实际应用示例

图是一种强大的数据结构，在解决现实世界中的许多问题时都能发挥重要作用。掌握图的相关知识和算法，对于提高程序设计能力和解决复杂问题都有很大帮助。

## 参考资源

1. Java数据结构与算法基础
2. 图论算法实践指南
3. 社交网络分析技术
4. 算法导论中的图算法章节