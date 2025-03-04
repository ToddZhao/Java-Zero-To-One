# Day 49: Java数据结构与算法 - 栈和队列

## 引言

栈和队列是两种最基本的线性数据结构，它们在计算机科学中有着广泛的应用。栈遵循后进先出（LIFO）原则，而队列遵循先进先出（FIFO）原则。本文将详细介绍这两种数据结构的实现方式和应用场景。

## 1. 栈（Stack）

### 1.1 栈的基本概念

栈是一种特殊的线性表，它只允许在一端（栈顶）进行插入和删除操作。栈的特点是：

- 后进先出（LIFO）
- 只能在栈顶操作
- 具有压栈（push）和出栈（pop）两种基本操作

### 1.2 栈的实现

```java
public class Stack<T> {
    private ArrayList<T> items;  // 使用ArrayList存储元素
    private int size;           // 栈的大小
    
    public Stack() {
        items = new ArrayList<>();
        size = 0;
    }
    
    // 判断栈是否为空
    public boolean isEmpty() {
        return size == 0;
    }
    
    // 获取栈的大小
    public int size() {
        return size;
    }
    
    // 入栈操作
    public void push(T item) {
        items.add(item);
        size++;
    }
    
    // 出栈操作
    public T pop() {
        if (isEmpty()) {
            throw new EmptyStackException();
        }
        T item = items.remove(size - 1);
        size--;
        return item;
    }
    
    // 查看栈顶元素
    public T peek() {
        if (isEmpty()) {
            throw new EmptyStackException();
        }
        return items.get(size - 1);
    }
}
```

### 1.3 栈的应用场景

1. 括号匹配检查

```java
public class BracketChecker {
    public static boolean isValid(String s) {
        Stack<Character> stack = new Stack<>();
        
        for (char c : s.toCharArray()) {
            if (c == '(' || c == '{' || c == '[') {
                stack.push(c);
            } else {
                if (stack.isEmpty()) return false;
                
                char top = stack.pop();
                if (c == ')' && top != '(') return false;
                if (c == '}' && top != '{') return false;
                if (c == ']' && top != '[') return false;
            }
        }
        
        return stack.isEmpty();
    }
}
```

2. 表达式求值

```java
public class ExpressionEvaluator {
    public static int evaluate(String expression) {
        Stack<Integer> numbers = new Stack<>();
        Stack<Character> operators = new Stack<>();
        
        for (char c : expression.toCharArray()) {
            if (Character.isDigit(c)) {
                numbers.push(c - '0');
            } else if (c == '+' || c == '-' || c == '*' || c == '/') {
                while (!operators.isEmpty() && 
                       hasPrecedence(operators.peek(), c)) {
                    numbers.push(applyOperation(operators.pop(), 
                                               numbers.pop(), 
                                               numbers.pop()));
                }
                operators.push(c);
            }
        }
        
        while (!operators.isEmpty()) {
            numbers.push(applyOperation(operators.pop(), 
                                       numbers.pop(), 
                                       numbers.pop()));
        }
        
        return numbers.pop();
    }
    
    private static boolean hasPrecedence(char op1, char op2) {
        return (op1 == '*' || op1 == '/') && (op2 == '+' || op2 == '-');
    }
    
    private static int applyOperation(char operator, int b, int a) {
        switch (operator) {
            case '+': return a + b;
            case '-': return a - b;
            case '*': return a * b;
            case '/': return a / b;
        }
        return 0;
    }
}
```

## 2. 队列（Queue）

### 2.1 队列的基本概念

队列是一种特殊的线性表，它只允许在一端（队尾）进行插入操作，在另一端（队头）进行删除操作。队列的特点是：

- 先进先出（FIFO）
- 只能在队尾插入，在队头删除
- 具有入队（enqueue）和出队（dequeue）两种基本操作

### 2.2 队列的实现

```java
public class Queue<T> {
    private LinkedList<T> items;  // 使用LinkedList存储元素
    private int size;             // 队列的大小
    
    public Queue() {
        items = new LinkedList<>();
        size = 0;
    }
    
    // 判断队列是否为空
    public boolean isEmpty() {
        return size == 0;
    }
    
    // 获取队列的大小
    public int size() {
        return size;
    }
    
    // 入队操作
    public void enqueue(T item) {
        items.addLast(item);
        size++;
    }
    
    // 出队操作
    public T dequeue() {
        if (isEmpty()) {
            throw new NoSuchElementException("Queue is empty");
        }
        size--;
        return items.removeFirst();
    }
    
    // 查看队首元素
    public T peek() {
        if (isEmpty()) {
            throw new NoSuchElementException("Queue is empty");
        }
        return items.getFirst();
    }
}
```

### 2.3 循环队列的实现

```java
public class CircularQueue<T> {
    private T[] items;           // 存储元素的数组
    private int front;           // 队首指针
    private int rear;            // 队尾指针
    private int size;            // 队列中的元素个数
    private static final int DEFAULT_CAPACITY = 10;
    
    @SuppressWarnings("unchecked")
    public CircularQueue() {
        items = (T[]) new Object[DEFAULT_CAPACITY];
        front = 0;
        rear = -1;
        size = 0;
    }
    
    // 判断队列是否为空
    public boolean isEmpty() {
        return size == 0;
    }
    
    // 判断队列是否已满
    public boolean isFull() {
        return size == items.length;
    }
    
    // 获取队列的大小
    public int size() {
        return size;
    }
    
    // 入队操作
    public void enqueue(T item) {
        if (isFull()) {
            throw new IllegalStateException("Queue is full");
        }
        rear = (rear + 1) % items.length;
        items[rear] = item;
        size++;
    }
    
    // 出队操作
    public T dequeue() {
        if (isEmpty()) {
            throw new NoSuchElementException("Queue is empty");
        }
        T item = items[front];
        items[front] = null;
        front = (front + 1) % items.length;
        size--;
        return item;
    }
    
    // 查看队首元素
    public T peek() {
        if (isEmpty()) {
            throw new NoSuchElementException("Queue is empty");
        }
        return items[front];
    }
}
```

### 2.4 队列的应用场景

1. 广度优先搜索（BFS）

```java
public class BFSExample {
    public static void bfs(Graph graph, int startVertex) {
        boolean[] visited = new boolean[graph.getVertexCount()];
        Queue<Integer> queue = new Queue<>();
        
        visited[startVertex] = true;
        queue.enqueue(startVertex);
        
        while (!queue.isEmpty()) {
            int vertex = queue.dequeue();
            System.out.print(vertex + " ");
            
            for (int adjacent : graph.getAdjacentVertices(vertex)) {
                if (!visited[adjacent]) {
                    visited[adjacent] = true;
                    queue.enqueue(adjacent);
                }
            }
        }
    }
}
```

2. 任务调度

```java
public class TaskScheduler {
    private Queue<Task> taskQueue;
    
    public TaskScheduler() {
        taskQueue = new Queue<>();
    }
    
    public void addTask(Task task) {
        taskQueue.enqueue(task);
    }
    
    public void executeTasks() {
        while (!taskQueue.isEmpty()) {
            Task task = taskQueue.dequeue();
            task.execute();
        }
    }
}
```

## 3. Java集合框架中的栈和队列

### 3.1 Stack类

Java提供了内置的Stack类：

```java
Stack<Integer> stack = new Stack<>();
stack.push(1);
stack.push(2);
int top = stack.pop();  // 返回2
```

### 3.2 Queue接口

Java提供了Queue接口及其实现类：

```java
Queue<String> queue = new LinkedList<>();
queue.offer("First");
queue.offer("Second");
String first = queue.poll();  // 返回"First"
```

## 总结

本文介绍了栈和队列这两种基本的数据结构，包括：

1. 栈的实现和应用
2. 队列的实现和应用
3. 循环队列的实现
4. Java集合框架中的相关类

这些数据结构在实际编程中有着广泛的应用，掌握它们的原理和实现方式对提高编程能力很有帮助。

## 参考资源

1. Java官方文档：https://docs.oracle.com/javase/tutorial/collections/
2. 数据结构与算法分析
3. Introduction to Algorithms (CLRS)
4. Java数据结构和算法实现指南