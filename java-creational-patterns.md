# Day 14: Java设计模式 - 创建型模式

## 引言

创建型设计模式关注对象的创建过程，它们提供了一种在创建对象时能够隐藏创建逻辑的方式，而不是直接使用new运算符实例化对象。本文将详细介绍Java中常用的创建型设计模式及其实践应用。

## 1. 单例模式（Singleton Pattern）

### 1.1 基本实现

```java
// 饿汉式单例
public class EagerSingleton {
    private static final EagerSingleton instance = new EagerSingleton();
    
    private EagerSingleton() {}
    
    public static EagerSingleton getInstance() {
        return instance;
    }
}

// 懒汉式单例（线程安全）
public class LazySingleton {
    private static volatile LazySingleton instance;
    
    private LazySingleton() {}
    
    public static LazySingleton getInstance() {
        if (instance == null) {
            synchronized (LazySingleton.class) {
                if (instance == null) {
                    instance = new LazySingleton();
                }
            }
        }
        return instance;
    }
}

// 静态内部类单例
public class StaticSingleton {
    private StaticSingleton() {}
    
    private static class SingletonHolder {
        private static final StaticSingleton INSTANCE = new StaticSingleton();
    }
    
    public static StaticSingleton getInstance() {
        return SingletonHolder.INSTANCE;
    }
}
```

## 2. 工厂模式（Factory Pattern）

### 2.1 简单工厂

```java
// 产品接口
public interface Product {
    void operation();
}

// 具体产品
public class ConcreteProductA implements Product {
    @Override
    public void operation() {
        System.out.println("ConcreteProductA operation");
    }
}

public class ConcreteProductB implements Product {
    @Override
    public void operation() {
        System.out.println("ConcreteProductB operation");
    }
}

// 简单工厂
public class SimpleFactory {
    public static Product createProduct(String type) {
        if ("A".equals(type)) {
            return new ConcreteProductA();
        } else if ("B".equals(type)) {
            return new ConcreteProductB();
        }
        throw new IllegalArgumentException("Unknown product type");
    }
}
```

### 2.2 工厂方法

```java
// 工厂接口
public interface Factory {
    Product createProduct();
}

// 具体工厂
public class ConcreteFactoryA implements Factory {
    @Override
    public Product createProduct() {
        return new ConcreteProductA();
    }
}

public class ConcreteFactoryB implements Factory {
    @Override
    public Product createProduct() {
        return new ConcreteProductB();
    }
}
```

## 3. 抽象工厂模式（Abstract Factory Pattern）

```java
// 抽象产品
public interface Button {}
public interface TextField {}

// 具体产品
public class WindowsButton implements Button {}
public class WindowsTextField implements TextField {}
public class MacButton implements Button {}
public class MacTextField implements TextField {}

// 抽象工厂
public interface GUIFactory {
    Button createButton();
    TextField createTextField();
}

// 具体工厂
public class WindowsFactory implements GUIFactory {
    @Override
    public Button createButton() {
        return new WindowsButton();
    }
    
    @Override
    public TextField createTextField() {
        return new WindowsTextField();
    }
}

public class MacFactory implements GUIFactory {
    @Override
    public Button createButton() {
        return new MacButton();
    }
    
    @Override
    public TextField createTextField() {
        return new MacTextField();
    }
}
```

## 4. 建造者模式（Builder Pattern）

```java
public class Computer {
    private String cpu;
    private String ram;
    private String storage;
    private String gpu;
    
    public static class Builder {
        private Computer computer = new Computer();
        
        public Builder cpu(String cpu) {
            computer.cpu = cpu;
            return this;
        }
        
        public Builder ram(String ram) {
            computer.ram = ram;
            return this;
        }
        
        public Builder storage(String storage) {
            computer.storage = storage;
            return this;
        }
        
        public Builder gpu(String gpu) {
            computer.gpu = gpu;
            return this;
        }
        
        public Computer build() {
            return computer;
        }
    }
}

// 使用示例
Computer computer = new Computer.Builder()
    .cpu("Intel i7")
    .ram("16GB")
    .storage("512GB SSD")
    .gpu("NVIDIA RTX 3070")
    .build();
```

## 5. 原型模式（Prototype Pattern）

```java
// 原型接口
public interface Prototype extends Cloneable {
    Prototype clone();
}

// 具体原型
public class ConcretePrototype implements Prototype {
    private String field;
    
    public ConcretePrototype(String field) {
        this.field = field;
    }
    
    @Override
    public Prototype clone() {
        try {
            return (Prototype) super.clone();
        } catch (CloneNotSupportedException e) {
            return null;
        }
    }
}
```

## 6. 实践案例

### 6.1 数据库连接池

```java
public class DatabaseConnectionPool {
    private static DatabaseConnectionPool instance;
    private List<Connection> connectionPool;
    
    private DatabaseConnectionPool() {
        connectionPool = new ArrayList<>();
        // 初始化连接池
    }
    
    public static synchronized DatabaseConnectionPool getInstance() {
        if (instance == null) {
            instance = new DatabaseConnectionPool();
        }
        return instance;
    }
    
    public synchronized Connection getConnection() {
        if (connectionPool.isEmpty()) {
            return createConnection();
        }
        return connectionPool.remove(connectionPool.size() - 1);
    }
    
    public synchronized void releaseConnection(Connection connection) {
        connectionPool.add(connection);
    }
}
```

### 6.2 GUI组件工厂

```java
public class GUIComponentFactory {
    private static final Map<String, Component> prototypeMap = new HashMap<>();
    
    static {
        prototypeMap.put("button", new JButton());
        prototypeMap.put("textField", new JTextField());
        prototypeMap.put("label", new JLabel());
    }
    
    public static Component createComponent(String type) {
        Component prototype = prototypeMap.get(type);
        if (prototype == null) {
            throw new IllegalArgumentException("Unknown component type");
        }
        return (Component) prototype.clone();
    }
}
```

## 7. 最佳实践

1. 选择合适的创建型模式
2. 注意线程安全问题
3. 避免过度使用设计模式
4. 考虑性能影响
5. 保持代码简洁可维护

## 总结

本文介绍了Java创建型设计模式的核心概念和实践应用，包括：

1. 单例模式的多种实现
2. 工厂模式的变体
3. 抽象工厂模式
4. 建造者模式
5. 原型模式

通过掌握这些模式，我们可以更好地组织对象创建相关的代码，提高代码的可维护性和可扩展性。

## 参考资源

1. 设计模式：可复用面向对象软件的基础
2. Head First设计模式
3. Java设计模式最佳实践
4. Spring框架中的设计模式应用