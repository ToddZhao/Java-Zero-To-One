# Day 15: Java设计模式 - 结构型模式

## 引言

结构型设计模式关注类和对象的组合，通过组合接口和实现，可以组成更大的结构。本文将详细介绍Java中常用的结构型设计模式及其实践应用。

## 1. 适配器模式（Adapter Pattern）

### 1.1 类适配器

```java
// 目标接口
public interface Target {
    void request();
}

// 被适配的类
public class Adaptee {
    public void specificRequest() {
        System.out.println("特殊请求");
    }
}

// 类适配器
public class ClassAdapter extends Adaptee implements Target {
    @Override
    public void request() {
        specificRequest();
    }
}
```

### 1.2 对象适配器

```java
// 对象适配器
public class ObjectAdapter implements Target {
    private Adaptee adaptee;
    
    public ObjectAdapter(Adaptee adaptee) {
        this.adaptee = adaptee;
    }
    
    @Override
    public void request() {
        adaptee.specificRequest();
    }
}
```

## 2. 装饰器模式（Decorator Pattern）

```java
// 组件接口
public interface Component {
    void operation();
}

// 具体组件
public class ConcreteComponent implements Component {
    @Override
    public void operation() {
        System.out.println("具体组件操作");
    }
}

// 装饰器
public abstract class Decorator implements Component {
    protected Component component;
    
    public Decorator(Component component) {
        this.component = component;
    }
    
    @Override
    public void operation() {
        component.operation();
    }
}

// 具体装饰器
public class ConcreteDecorator extends Decorator {
    public ConcreteDecorator(Component component) {
        super(component);
    }
    
    @Override
    public void operation() {
        super.operation();
        addedBehavior();
    }
    
    private void addedBehavior() {
        System.out.println("增加的行为");
    }
}
```

## 3. 代理模式（Proxy Pattern）

```java
// 主题接口
public interface Subject {
    void request();
}

// 真实主题
public class RealSubject implements Subject {
    @Override
    public void request() {
        System.out.println("真实主题的请求");
    }
}

// 代理
public class Proxy implements Subject {
    private RealSubject realSubject;
    
    @Override
    public void request() {
        if (realSubject == null) {
            realSubject = new RealSubject();
        }
        preRequest();
        realSubject.request();
        postRequest();
    }
    
    private void preRequest() {
        System.out.println("请求前的处理");
    }
    
    private void postRequest() {
        System.out.println("请求后的处理");
    }
}
```

## 4. 外观模式（Facade Pattern）

```java
// 子系统类
public class SubSystemOne {
    public void methodOne() {
        System.out.println("子系统方法一");
    }
}

public class SubSystemTwo {
    public void methodTwo() {
        System.out.println("子系统方法二");
    }
}

// 外观类
public class Facade {
    private SubSystemOne one;
    private SubSystemTwo two;
    
    public Facade() {
        one = new SubSystemOne();
        two = new SubSystemTwo();
    }
    
    public void methodA() {
        System.out.println("方法组A");
        one.methodOne();
        two.methodTwo();
    }
    
    public void methodB() {
        System.out.println("方法组B");
        two.methodTwo();
    }
}
```

## 5. 桥接模式（Bridge Pattern）

```java
// 实现接口
public interface Implementor {
    void operationImpl();
}

// 具体实现
public class ConcreteImplementorA implements Implementor {
    @Override
    public void operationImpl() {
        System.out.println("具体实现A");
    }
}

// 抽象类
public abstract class Abstraction {
    protected Implementor implementor;
    
    public Abstraction(Implementor implementor) {
        this.implementor = implementor;
    }
    
    public abstract void operation();
}

// 修正抽象类
public class RefinedAbstraction extends Abstraction {
    public RefinedAbstraction(Implementor implementor) {
        super(implementor);
    }
    
    @Override
    public void operation() {
        System.out.println("修正抽象类的操作");
        implementor.operationImpl();
    }
}
```

## 6. 组合模式（Composite Pattern）

```java
public abstract class Component {
    protected String name;
    
    public Component(String name) {
        this.name = name;
    }
    
    public abstract void add(Component c);
    public abstract void remove(Component c);
    public abstract void display(int depth);
}

// 叶子节点
public class Leaf extends Component {
    public Leaf(String name) {
        super(name);
    }
    
    @Override
    public void add(Component c) {
        System.out.println("Cannot add to a leaf");
    }
    
    @Override
    public void remove(Component c) {
        System.out.println("Cannot remove from a leaf");
    }
    
    @Override
    public void display(int depth) {
        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < depth; i++) {
            sb.append("-");
        }
        System.out.println(sb.toString() + name);
    }
}

// 复合节点
public class Composite extends Component {
    private List<Component> children = new ArrayList<>();
    
    public Composite(String name) {
        super(name);
    }
    
    @Override
    public void add(Component c) {
        children.add(c);
    }
    
    @Override
    public void remove(Component c) {
        children.remove(c);
    }
    
    @Override
    public void display(int depth) {
        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < depth; i++) {
            sb.append("-");
        }
        System.out.println(sb.toString() + name);
        
        for (Component component : children) {
            component.display(depth + 2);
        }
    }
}
```

## 7. 享元模式（Flyweight Pattern）

```java
// 享元接口
public interface Flyweight {
    void operation(String extrinsicState);
}

// 具体享元
public class ConcreteFlyweight implements Flyweight {
    private String intrinsicState;
    
    public ConcreteFlyweight(String intrinsicState) {
        this.intrinsicState = intrinsicState;
    }
    
    @Override
    public void operation(String extrinsicState) {
        System.out.println("内部状态: " + intrinsicState);
        System.out.println("外部状态: " + extrinsicState);
    }
}

// 享元工厂
public class FlyweightFactory {
    private Map<String, Flyweight> flyweights = new HashMap<>();
    
    public Flyweight getFlyweight(String key) {
        if (!flyweights.containsKey(key)) {
            flyweights.put(key, new ConcreteFlyweight(key));
        }
        return flyweights.get(key);
    }
}
```

## 8. 实践案例

### 8.1 日志装饰器

```java
public interface Logger {
    void log(String message);
}

public class BasicLogger implements Logger {
    @Override
    public void log(String message) {
        System.out.println(message);
    }
}

public class TimestampDecorator implements Logger {
    private Logger logger;
    
    public TimestampDecorator(Logger logger) {
        this.logger = logger;
    }
    
    @Override
    public void log(String message) {
        String timestampMessage = 
            LocalDateTime.now() + ": " + message;
        logger.log(timestampMessage);
    }
}
```

### 8.2 缓存代理

```java
public interface DataService {
    String getData(String key);
}

public class RealDataService implements DataService {
    @Override
    public String getData(String key) {
        // 模拟耗时操作
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return "Data for " + key;
    }
}

public class CacheProxy implements DataService {
    private RealDataService realService;
    private Map<String, String> cache;
    
    public CacheProxy() {
        this.realService = new RealDataService();
        this.cache = new HashMap<>();
    }
    
    @Override
    public String getData(String key) {
        if (cache.containsKey(key)) {
            System.out.println("从缓存获取数据");
            return cache.get(key);
        }
        
        String data = realService.getData(key);
        cache.put(key, data);
        return data;
    }
}
```

## 9. 最佳实践

1. 根据实际需求选择合适的模式
2. 避免过度设计
3. 注意模式的适用场景
4. 保持代码的可维护性
5. 考虑性能影响

## 总结

本文介绍了Java结构型设计模式的核心概念和实践应用，包括：

1. 适配器模式
2. 装饰器模式
3. 代理模式
4. 外观模式
5. 桥接模式
6. 组合模式
7. 享元模式

通过掌握这些模式，我们可以更好地组织类和对象的结构，提高代码的复用性和灵活性。

## 参考资源

1. 设计模式：可复用面向对象软件的基础
2. Head First设计模式
3. Java设计模式最佳实践
4. Spring框架中的设计模式应用