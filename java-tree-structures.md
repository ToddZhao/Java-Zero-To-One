# Day 50: Java数据结构与算法 - 树

## 引言

树是计算机科学中最重要的非线性数据结构之一，它模拟了具有层次关系的数据集合。与线性数据结构（如数组、链表、栈和队列）不同，树是一种分层数据模型，广泛应用于文件系统、数据库索引、语法分析等领域。本文将深入介绍树的概念、类型、实现方式以及在Java中的应用。

## 1. 树的基础概念

### 1.1 什么是树

树是由节点和边组成的连通无环图，它具有以下特点：

- 每个节点都有零个或多个子节点
- 除了根节点外，每个节点都有且只有一个父节点
- 没有环路
- 所有节点都可以通过根节点到达

### 1.2 树的术语

- **节点（Node）**：树中的基本单位，包含数据和指向子节点的引用
- **根节点（Root）**：树的顶层节点，没有父节点
- **叶节点（Leaf）**：没有子节点的节点
- **父节点（Parent）**：直接连接到子节点的节点
- **子节点（Child）**：直接连接到父节点的节点
- **兄弟节点（Sibling）**：共享同一个父节点的节点
- **深度（Depth）**：从根节点到指定节点的边数
- **高度（Height）**：从指定节点到最远叶节点的边数
- **层（Level）**：节点的深度加1

## 2. 二叉树

### 2.1 二叉树的定义

二叉树是每个节点最多有两个子节点的树，通常称为左子节点和右子节点。

### 2.2 二叉树节点的实现

```java
public class TreeNode<T> {
    private T data;
    private TreeNode<T> left;
    private TreeNode<T> right;
    
    public TreeNode(T data) {
        this.data = data;
        this.left = null;
        this.right = null;
    }
    
    // Getters and setters
    public T getData() {
        return data;
    }
    
    public void setData(T data) {
        this.data = data;
    }
    
    public TreeNode<T> getLeft() {
        return left;
    }
    
    public void setLeft(TreeNode<T> left) {
        this.left = left;
    }
    
    public TreeNode<T> getRight() {
        return right;
    }
    
    public void setRight(TreeNode<T> right) {
        this.right = right;
    }
}
```

### 2.3 二叉树的实现

```java
public class BinaryTree<T> {
    private TreeNode<T> root;
    
    public BinaryTree() {
        this.root = null;
    }
    
    public BinaryTree(T rootData) {
        this.root = new TreeNode<>(rootData);
    }
    
    public TreeNode<T> getRoot() {
        return root;
    }
    
    public void setRoot(TreeNode<T> root) {
        this.root = root;
    }
    
    // 判断树是否为空
    public boolean isEmpty() {
        return root == null;
    }
    
    // 计算树的高度
    public int height() {
        return height(root);
    }
    
    private int height(TreeNode<T> node) {
        if (node == null) {
            return -1;
        }
        return 1 + Math.max(height(node.getLeft()), height(node.getRight()));
    }
    
    // 计算节点数
    public int size() {
        return size(root);
    }
    
    private int size(TreeNode<T> node) {
        if (node == null) {
            return 0;
        }
        return 1 + size(node.getLeft()) + size(node.getRight());
    }
}
```

## 3. 二叉树的遍历

### 3.1 前序遍历（根-左-右）

```java
public void preOrderTraversal() {
    preOrderTraversal(root);
}

private void preOrderTraversal(TreeNode<T> node) {
    if (node == null) {
        return;
    }
    
    System.out.print(node.getData() + " "); // 访问根节点
    preOrderTraversal(node.getLeft());     // 遍历左子树
    preOrderTraversal(node.getRight());    // 遍历右子树
}
```

### 3.2 中序遍历（左-根-右）

```java
public void inOrderTraversal() {
    inOrderTraversal(root);
}

private void inOrderTraversal(TreeNode<T> node) {
    if (node == null) {
        return;
    }
    
    inOrderTraversal(node.getLeft());      // 遍历左子树
    System.out.print(node.getData() + " "); // 访问根节点
    inOrderTraversal(node.getRight());     // 遍历右子树
}
```

### 3.3 后序遍历（左-右-根）

```java
public void postOrderTraversal() {
    postOrderTraversal(root);
}

private void postOrderTraversal(TreeNode<T> node) {
    if (node == null) {
        return;
    }
    
    postOrderTraversal(node.getLeft());    // 遍历左子树
    postOrderTraversal(node.getRight());   // 遍历右子树
    System.out.print(node.getData() + " "); // 访问根节点
}
```

### 3.4 层序遍历（广度优先）

```java
public void levelOrderTraversal() {
    if (root == null) {
        return;
    }
    
    Queue<TreeNode<T>> queue = new LinkedList<>();
    queue.offer(root);
    
    while (!queue.isEmpty()) {
        TreeNode<T> current = queue.poll();
        System.out.print(current.getData() + " ");
        
        if (current.getLeft() != null) {
            queue.offer(current.getLeft());
        }
        
        if (current.getRight() != null) {
            queue.offer(current.getRight());
        }
    }
}
```

## 4. 二叉搜索树（BST）

### 4.1 二叉搜索树的定义

二叉搜索树是一种特殊的二叉树，它具有以下性质：

- 左子树上所有节点的值都小于根节点的值
- 右子树上所有节点的值都大于根节点的值
- 左右子树也都是二叉搜索树

### 4.2 二叉搜索树的实现

```java
public class BinarySearchTree<T extends Comparable<T>> {
    private TreeNode<T> root;
    
    public BinarySearchTree() {
        this.root = null;
    }
    
    // 插入节点
    public void insert(T data) {
        root = insert(root, data);
    }
    
    private TreeNode<T> insert(TreeNode<T> node, T data) {
        if (node == null) {
            return new TreeNode<>(data);
        }
        
        int compareResult = data.compareTo(node.getData());
        
        if (compareResult < 0) {
            node.setLeft(insert(node.getLeft(), data));
        } else if (compareResult > 0) {
            node.setRight(insert(node.getRight(), data));
        }
        // 如果相等，不做任何操作（不允许重复值）
        
        return node;
    }
    
    // 查找节点
    public boolean contains(T data) {
        return contains(root, data);
    }
    
    private boolean contains(TreeNode<T> node, T data) {
        if (node == null) {
            return false;
        }
        
        int compareResult = data.compareTo(node.getData());
        
        if (compareResult < 0) {
            return contains(node.getLeft(), data);
        } else if (compareResult > 0) {
            return contains(node.getRight(), data);
        } else {
            return true; // 找到了
        }
    }
    
    // 删除节点
    public void remove(T data) {
        root = remove(root, data);
    }
    
    private TreeNode<T> remove(TreeNode<T> node, T data) {
        if (node == null) {
            return null;
        }
        
        int compareResult = data.compareTo(node.getData());
        
        if (compareResult < 0) {
            node.setLeft(remove(node.getLeft(), data));
        } else if (compareResult > 0) {
            node.setRight(remove(node.getRight(), data));
        } else {
            // 找到要删除的节点
            
            // 情况1：叶节点（没有子节点）
            if (node.getLeft() == null && node.getRight() == null) {
                return null;
            }
            
            // 情况2：只有一个子节点
            if (node.getLeft() == null) {
                return node.getRight();
            }
            if (node.getRight() == null) {
                return node.getLeft();
            }
            
            // 情况3：有两个子节点
            // 找到右子树中的最小值作为后继
            node.setData(findMin(node.getRight()));
            // 删除右子树中的最小值节点
            node.setRight(remove(node.getRight(), node.getData()));
        }
        
        return node;
    }
    
    // 查找最小值
    public T findMin() {
        if (isEmpty()) {
            throw new NoSuchElementException("Tree is empty");
        }
        return findMin(root);
    }
    
    private T findMin(TreeNode<T> node) {
        if (node.getLeft() == null) {
            return node.getData();
        }
        return findMin(node.getLeft());
    }
    
    // 查找最大值
    public T findMax() {
        if (isEmpty()) {
            throw new NoSuchElementException("Tree is empty");
        }
        return findMax(root);
    }
    
    private T findMax(TreeNode<T> node) {
        if (node.getRight() == null) {
            return node.getData();
        }
        return findMax(node.getRight());
    }
    
    // 判断树是否为空
    public boolean isEmpty() {
        return root == null;
    }
    
    // 中序遍历（按顺序输出）
    public void inOrderTraversal() {
        inOrderTraversal(root);
        System.out.println();
    }
    
    private void inOrderTraversal(TreeNode<T> node) {
        if (node == null) {
            return;
        }
        
        inOrderTraversal(node.getLeft());
        System.out.print(node.getData() + " ");
        inOrderTraversal(node.getRight());
    }
}
```

## 5. 平衡二叉树（AVL树）

### 5.1 AVL树的定义

AVL树是一种自平衡的二叉搜索树，它确保树的高度保持在O(log n)，从而保证搜索、插入和删除操作的时间复杂度为O(log n)。

### 5.2 AVL树节点的实现

```java
public class AVLNode<T extends Comparable<T>> {
    private T data;
    private AVLNode<T> left;
    private AVLNode<T> right;
    private int height;
    
    public AVLNode(T data) {
        this.data = data;
        this.left = null;
        this.right = null;
        this.height = 0;
    }
    
    // Getters and setters
    public T getData() {
        return data;
    }
    
    public void setData(T data) {
        this.data = data;
    }
    
    public AVLNode<T> getLeft() {
        return left;
    }
    
    public void setLeft(AVLNode<T> left) {
        this.left = left;
    }
    
    public AVLNode<T> getRight() {
        return right;
    }
    
    public void setRight(AVLNode<T> right) {
        this.right = right;
    }
    
    public int getHeight() {
        return height;
    }
    
    public void setHeight(int height) {
        this.height = height;
    }
}
```

### 5.3 AVL树的实现

```java
public class AVLTree<T extends