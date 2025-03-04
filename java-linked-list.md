# Day 48: Java数据结构与算法 - 链表

## 引言

链表是计算机科学中最基础、最常用的数据结构之一。与数组不同，链表是一种动态的数据结构，它通过节点之间的引用连接而成，可以高效地进行插入和删除操作。本文将深入介绍链表的概念、类型、实现方式以及在Java中的应用。

## 1. 链表基础

### 1.1 什么是链表

链表是由一系列节点组成的数据结构，每个节点包含两部分：数据域和指针域。数据域存储实际的数据，指针域存储下一个节点的地址。

### 1.2 链表的类型

- **单向链表**：每个节点只有一个指向下一个节点的指针
- **双向链表**：每个节点有两个指针，分别指向前一个和后一个节点
- **循环链表**：最后一个节点指向第一个节点，形成一个环

## 2. 单向链表的实现

### 2.1 节点类定义

```java
public class Node<T> {
    private T data;          // 数据域
    private Node<T> next;    // 指针域
    
    public Node(T data) {
        this.data = data;
        this.next = null;
    }
    
    public T getData() {
        return data;
    }
    
    public void setData(T data) {
        this.data = data;
    }
    
    public Node<T> getNext() {
        return next;
    }
    
    public void setNext(Node<T> next) {
        this.next = next;
    }
}
```

### 2.2 单向链表类实现

```java
public class SinglyLinkedList<T> {
    private Node<T> head;  // 头节点
    private int size;      // 链表大小
    
    public SinglyLinkedList() {
        this.head = null;
        this.size = 0;
    }
    
    // 判断链表是否为空
    public boolean isEmpty() {
        return head == null;
    }
    
    // 获取链表大小
    public int size() {
        return size;
    }
    
    // 在链表头部添加元素
    public void addFirst(T data) {
        Node<T> newNode = new Node<>(data);
        if (isEmpty()) {
            head = newNode;
        } else {
            newNode.setNext(head);
            head = newNode;
        }
        size++;
    }
    
    // 在链表尾部添加元素
    public void addLast(T data) {
        Node<T> newNode = new Node<>(data);
        if (isEmpty()) {
            head = newNode;
        } else {
            Node<T> current = head;
            while (current.getNext() != null) {
                current = current.getNext();
            }
            current.setNext(newNode);
        }
        size++;
    }
    
    // 在指定位置添加元素
    public void add(int index, T data) {
        if (index < 0 || index > size) {
            throw new IndexOutOfBoundsException("Index: " + index + ", Size: " + size);
        }
        
        if (index == 0) {
            addFirst(data);
            return;
        }
        
        Node<T> newNode = new Node<>(data);
        Node<T> current = head;
        for (int i = 0; i < index - 1; i++) {
            current = current.getNext();
        }
        newNode.setNext(current.getNext());
        current.setNext(newNode);
        size++;
    }
    
    // 删除第一个元素
    public T removeFirst() {
        if (isEmpty()) {
            throw new NoSuchElementException("List is empty");
        }
        
        T data = head.getData();
        head = head.getNext();
        size--;
        return data;
    }
    
    // 删除最后一个元素
    public T removeLast() {
        if (isEmpty()) {
            throw new NoSuchElementException("List is empty");
        }
        
        if (head.getNext() == null) {
            T data = head.getData();
            head = null;
            size--;
            return data;
        }
        
        Node<T> current = head;
        while (current.getNext().getNext() != null) {
            current = current.getNext();
        }
        
        T data = current.getNext().getData();
        current.setNext(null);
        size--;
        return data;
    }
    
    // 删除指定位置的元素
    public T remove(int index) {
        if (isEmpty()) {
            throw new NoSuchElementException("List is empty");
        }
        
        if (index < 0 || index >= size) {
            throw new IndexOutOfBoundsException("Index: " + index + ", Size: " + size);
        }
        
        if (index == 0) {
            return removeFirst();
        }
        
        Node<T> current = head;
        for (int i = 0; i < index - 1; i++) {
            current = current.getNext();
        }
        
        T data = current.getNext().getData();
        current.setNext(current.getNext().getNext());
        size--;
        return data;
    }
    
    // 获取指定位置的元素
    public T get(int index) {
        if (index < 0 || index >= size) {
            throw new IndexOutOfBoundsException("Index: " + index + ", Size: " + size);
        }
        
        Node<T> current = head;
        for (int i = 0; i < index; i++) {
            current = current.getNext();
        }
        
        return current.getData();
    }
    
    // 查找元素的索引
    public int indexOf(T data) {
        Node<T> current = head;
        int index = 0;
        
        while (current != null) {
            if (current.getData().equals(data)) {
                return index;
            }
            current = current.getNext();
            index++;
        }
        
        return -1;  // 未找到元素
    }
    
    // 打印链表
    public void printList() {
        Node<T> current = head;
        System.out.print("LinkedList: ");
        while (current != null) {
            System.out.print(current.getData() + " -> ");
            current = current.getNext();
        }
        System.out.println("null");
    }
}
```

## 3. 双向链表的实现

### 3.1 双向链表节点类

```java
public class DoublyNode<T> {
    private T data;
    private DoublyNode<T> prev;
    private DoublyNode<T> next;
    
    public DoublyNode(T data) {
        this.data = data;
        this.prev = null;
        this.next = null;
    }
    
    // Getters and setters
    public T getData() {
        return data;
    }
    
    public void setData(T data) {
        this.data = data;
    }
    
    public DoublyNode<T> getPrev() {
        return prev;
    }
    
    public void setPrev(DoublyNode<T> prev) {
        this.prev = prev;
    }
    
    public DoublyNode<T> getNext() {
        return next;
    }
    
    public void setNext(DoublyNode<T> next) {
        this.next = next;
    }
}
```

### 3.2 双向链表类实现

```java
public class DoublyLinkedList<T> {
    private DoublyNode<T> head;
    private DoublyNode<T> tail;
    private int size;
    
    public DoublyLinkedList() {
        this.head = null;
        this.tail = null;
        this.size = 0;
    }
    
    // 判断链表是否为空
    public boolean isEmpty() {
        return head == null;
    }
    
    // 获取链表大小
    public int size() {
        return size;
    }
    
    // 在链表头部添加元素
    public void addFirst(T data) {
        DoublyNode<T> newNode = new DoublyNode<>(data);
        if (isEmpty()) {
            head = newNode;
            tail = newNode;
        } else {
            newNode.setNext(head);
            head.setPrev(newNode);
            head = newNode;
        }
        size++;
    }
    
    // 在链表尾部添加元素
    public void addLast(T data) {
        DoublyNode<T> newNode = new DoublyNode<>(data);
        if (isEmpty()) {
            head = newNode;
            tail = newNode;
        } else {
            tail.setNext(newNode);
            newNode.setPrev(tail);
            tail = newNode;
        }
        size++;
    }
    
    // 删除第一个元素
    public T removeFirst() {
        if (isEmpty()) {
            throw new NoSuchElementException("List is empty");
        }
        
        T data = head.getData();
        
        if (head == tail) {
            head = null;
            tail = null;
        } else {
            head = head.getNext();
            head.setPrev(null);
        }
        
        size--;
        return data;
    }
    
    // 删除最后一个元素
    public T removeLast() {
        if (isEmpty()) {
            throw new NoSuchElementException("List is empty");
        }
        
        T data = tail.getData();
        
        if (head == tail) {
            head = null;
            tail = null;
        } else {
            tail = tail.getPrev();
            tail.setNext(null);
        }
        
        size--;
        return data;
    }
    
    // 打印链表（从头到尾）
    public void printForward() {
        DoublyNode<T> current = head;
        System.out.print("Forward: ");
        while (current != null) {
            System.out.print(current.getData() + " <-> ");
            current = current.getNext();
        }
        System.out.println("null");
    }
    
    // 打印链表（从尾到头）
    public void printBackward() {
        DoublyNode<T> current = tail;
        System.out.print("Backward: ");
        while (current != null) {
            System.out.print(current.getData() + " <-> ");
            current = current.getPrev();
        }
        System.out.println("null");
    }
}
```

## 4. 链表的常见操作和算法

### 4.1 链表反转

```java
public void reverse() {
    if (head == null || head.getNext() == null) {
        return;  // 空链表或只有一个节点，无需反转
    }
    
    Node<T> prev = null;
    Node<T> current = head;
    Node<T> next = null;
    
    while (current != null) {
        next = current.getNext();  // 保存下一个节点
        current.setNext(prev);    // 反转当前节点的指针
        prev = current;           // 移动prev指针
        current = next;           // 移动current指针
    }
    
    head = prev;  // 更新头节点
}
```

### 4.2 检测环

```java
public boolean hasCycle() {
    if (head == null || head.getNext() == null) {
        return false;
    }
    
    Node<T> slow = head;
    Node<T> fast = head;
    
    while (fast != null && fast.getNext() != null) {
        slow = slow.getNext();          // 慢指针每次移动一步
        fast = fast.getNext().getNext(); // 快指针每次移动两步
        
        if (slow == fast) {  // 如果两个指针相遇，说明有环
            return true;
        }
    }
    
    return false;  // 如果fast指针到达链表尾部，说明没有环
}
```

### 4.3 查找中间节点

```java
public T findMiddle() {
    if (head == null) {
        throw new NoSuchElementException("List is empty");
    }
    
    Node<T> slow = head;
    Node<T> fast = head;
    
    while (fast != null && fast.getNext() != null) {
        slow = slow.getNext();          // 慢指针每次移动一步
        fast = fast.getNext().getNext(); // 快指针每次移动两步
    }
    
    return slow.getData();  // 当fast到达链表尾部时，slow正好在中间
}
```

## 5. 链表与集合框架