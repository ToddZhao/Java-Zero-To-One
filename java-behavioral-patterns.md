# Day 16: Java设计模式 - 行为型模式

## 引言

行为型设计模式关注对象之间的通信，它们定义了对象之间的交互方式和职责分配。本文将详细介绍Java中常用的行为型设计模式及其实践应用。

## 1. 观察者模式（Observer Pattern）

```java
// 观察者接口
public interface Observer {
    void update(String message);
}

// 具体观察者
public class ConcreteObserver implements Observer {
    private String name;
    
    public ConcreteObserver(String name) {
        this.name = name;
    }
    
    @Override
    public void update(String message) {
        System.out.println(name + "收到消息：" + message);
    }
}

// 主题接口
public interface Subject {
    void registerObserver(Observer observer);
    void removeObserver(Observer observer);
    void notifyObservers(String message);
}

// 具体主题
public class ConcreteSubject implements Subject {
    private List<Observer> observers = new ArrayList<>();
    
    @Override
    public void registerObserver(Observer observer) {
        observers.add(observer);
    }
    
    @Override
    public void removeObserver(Observer observer) {
        observers.remove(observer);
    }
    
    @Override
    public void notifyObservers(String message) {
        for (Observer observer : observers) {
            observer.update(message);
        }
    }
}
```

## 2. 策略模式（Strategy Pattern）

```java
// 策略接口
public interface PaymentStrategy {
    void pay(int amount);
}

// 具体策略
public class CreditCardPayment implements PaymentStrategy {
    @Override
    public void pay(int amount) {
        System.out.println("使用信用卡支付：" + amount);
    }
}

public class AlipayPayment implements PaymentStrategy {
    @Override
    public void pay(int amount) {
        System.out.println("使用支付宝支付：" + amount);
    }
}

// 上下文
public class PaymentContext {
    private PaymentStrategy strategy;
    
    public void setStrategy(PaymentStrategy strategy) {
        this.strategy = strategy;
    }
    
    public void executePayment(int amount) {
        strategy.pay(amount);
    }
}
```

## 3. 命令模式（Command Pattern）

```java
// 命令接口
public interface Command {
    void execute();
}

// 接收者
public class Light {
    public void turnOn() {
        System.out.println("开灯");
    }
    
    public void turnOff() {
        System.out.println("关灯");
    }
}

// 具体命令
public class LightOnCommand implements Command {
    private Light light;
    
    public LightOnCommand(Light light) {
        this.light = light;
    }
    
    @Override
    public void execute() {
        light.turnOn();
    }
}

// 调用者
public class RemoteControl {
    private Command command;
    
    public void setCommand(Command command) {
        this.command = command;
    }
    
    public void pressButton() {
        command.execute();
    }
}
```

## 4. 模板方法模式（Template Method Pattern）

```java
public abstract class DataMiner {
    // 模板方法
    public final void mine() {
        openFile();
        extractData();
        parseData();
        analyzeData();
        sendReport();
        closeFile();
    }
    
    abstract void openFile();
    abstract void extractData();
    
    void parseData() {
        System.out.println("解析数据");
    }
    
    void analyzeData() {
        System.out.println("分析数据");
    }
    
    void sendReport() {
        System.out.println("发送报告");
    }
    
    void closeFile() {
        System.out.println("关闭文件");
    }
}

// 具体类
public class PDFDataMiner extends DataMiner {
    @Override
    void openFile() {
        System.out.println("打开PDF文件");
    }
    
    @Override
    void extractData() {
        System.out.println("从PDF提取数据");
    }
}
```

## 5. 状态模式（State Pattern）

```java
// 状态接口
public interface State {
    void handle(Context context);
}

// 具体状态
public class ConcreteStateA implements State {
    @Override
    public void handle(Context context) {
        System.out.println("处理状态A");
        context.setState(new ConcreteStateB());
    }
}

public class ConcreteStateB implements State {
    @Override
    public void handle(Context context) {
        System.out.println("处理状态B");
        context.setState(new ConcreteStateA());
    }
}

// 上下文
public class Context {
    private State state;
    
    public Context() {
        state = new ConcreteStateA();
    }
    
    public void setState(State state) {
        this.state = state;
    }
    
    public void request() {
        state.handle(this);
    }
}
```

## 6. 责任链模式（Chain of Responsibility Pattern）

```java
public abstract class Handler {
    protected Handler successor;
    
    public void setSuccessor(Handler successor) {
        this.successor = successor;
    }
    
    public abstract void handleRequest(Request request);
}

public class ConcreteHandlerA extends Handler {
    @Override
    public void handleRequest(Request request) {
        if (request.getType().equals("TypeA")) {
            System.out.println("处理TypeA请求");
        } else if (successor != null) {
            successor.handleRequest(request);
        }
    }
}

public class ConcreteHandlerB extends Handler {
    @Override
    public void handleRequest(Request request) {
        if (request.getType().equals("TypeB")) {
            System.out.println("处理TypeB请求");
        } else if (successor != null) {
            successor.handleRequest(request);
        }
    }
}
```

## 7. 实践案例

### 7.1 日志记录器

```java
public abstract class LoggerChain {
    protected LoggerChain nextLogger;
    protected int level;
    
    public void setNextLogger(LoggerChain nextLogger) {
        this.nextLogger = nextLogger;
    }
    
    public void logMessage(int level, String message) {
        if (this.level <= level) {
            write(message);
        }
        if (nextLogger != null) {
            nextLogger.logMessage(level, message);
        }
    }
    
    abstract protected void write(String message);
}

public class ConsoleLogger extends LoggerChain {
    public ConsoleLogger(int level) {
        this.level = level;
    }
    
    @Override
    protected void write(String message) {
        System.out.println("Console::Logger: " + message);
    }
}

public class FileLogger extends LoggerChain {
    public FileLogger(int level) {
        this.level = level;
    }
    
    @Override
    protected void write(String message) {
        System.out.println("File::Logger: " + message);
    }
}
```

### 7.2 支付处理

```java
public interface PaymentHandler {
    void setNext(PaymentHandler next);
    void process(Payment payment);
}

public class PaymentValidationHandler implements PaymentHandler {
    private PaymentHandler next;
    
    @Override
    public void setNext(PaymentHandler next) {
        this.next = next;
    }
    
    @Override
    public void process(Payment payment) {
        if (payment.getAmount() <= 0) {
            throw new IllegalArgumentException("Invalid payment amount");
        }
        
        if (next != null) {
            next.process(payment);
        }
    }
}

public class PaymentProcessingHandler implements PaymentHandler {
    private PaymentHandler next;
    
    @Override
    public void setNext(PaymentHandler next) {
        this.next = next;
    }
    
    @Override
    public void process(Payment payment) {
        // 处理支付
        System.out.println("Processing payment: " + payment.getAmount());
        
        if (next != null) {
            next.process(payment);
        }
    }
}
```

## 8. 最佳实践

1. 选择合适的行为模式
2. 注意模式的使用场景
3. 避免过度设计
4. 保持代码简洁清晰
5. 考虑扩展性和维护性

## 总结

本文介绍了Java行为型设计模式的核心概念和实践应用，包括：

1. 观察者模式
2. 策略模式
3. 命令模式
4. 模板方法模式
5. 状态模式
6. 责任链模式

通过掌握这些模式，我们可以更好地处理对象之间的交互，提高代码的灵活性和可维护性。

## 参考资源

1. 设计模式：可复用面向对象软件的基础
2. Head First设计模式
3. Java设计模式最佳实践
4. Spring框架中的设计模式应用