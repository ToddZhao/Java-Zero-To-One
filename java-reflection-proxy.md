# Day 6: Java反射和动态代理

## 引言

Java反射和动态代理是Java语言中非常强大的特性，它们使得我们能够在运行时检查和操作类、接口、字段和方法。这些特性在框架开发、依赖注入、AOP等场景中被广泛应用。本文将深入介绍Java反射和动态代理的核心概念和实践应用。

## 1. Java反射基础

### 1.1 什么是反射

反射是Java程序在运行时检查、访问和修改自身状态或行为的能力。通过反射，我们可以：

- 获取类的信息
- 创建类的实例
- 访问和修改字段
- 调用方法
- 获取注解信息

### 1.2 获取Class对象

```java
// 方式1：通过类名
Class<?> class1 = String.class;

// 方式2：通过对象
String str = "Hello";
Class<?> class2 = str.getClass();

// 方式3：通过完整类名
Class<?> class3 = Class.forName("java.lang.String");
```

## 2. 反射的应用

### 2.1 获取类信息

```java
public class ReflectionExample {
    public static void printClassInfo(Class<?> clazz) {
        System.out.println("类名：" + clazz.getName());
        System.out.println("简单类名：" + clazz.getSimpleName());
        System.out.println("包名：" + clazz.getPackage().getName());
        System.out.println("是否为接口：" + clazz.isInterface());
        System.out.println("父类：" + clazz.getSuperclass().getName());
        
        // 获取所有公共方法
        Method[] methods = clazz.getMethods();
        System.out.println("\n公共方法：");
        for (Method method : methods) {
            System.out.println(method.getName());
        }
        
        // 获取所有公共字段
        Field[] fields = clazz.getFields();
        System.out.println("\n公共字段：");
        for (Field field : fields) {
            System.out.println(field.getName());
        }
    }
}
```

### 2.2 创建实例和调用方法

```java
public class ReflectionInvocation {
    public static void main(String[] args) throws Exception {
        // 获取类对象
        Class<?> clazz = Class.forName("com.example.MyClass");
        
        // 创建实例
        Object obj = clazz.newInstance();
        
        // 获取方法
        Method method = clazz.getMethod("myMethod", String.class);
        
        // 调用方法
        Object result = method.invoke(obj, "Hello");
    }
}
```

## 3. 动态代理

### 3.1 什么是动态代理

动态代理是一种在运行时创建代理类的机制，可以在不修改原有代码的情况下，添加额外的功能。

### 3.2 JDK动态代理

```java
// 接口定义
public interface UserService {
    void addUser(String name);
    void deleteUser(String name);
}

// 实现类
public class UserServiceImpl implements UserService {
    @Override
    public void addUser(String name) {
        System.out.println("添加用户：" + name);
    }
    
    @Override
    public void deleteUser(String name) {
        System.out.println("删除用户：" + name);
    }
}

// 代理处理器
public class LogHandler implements InvocationHandler {
    private final Object target;
    
    public LogHandler(Object target) {
        this.target = target;
    }
    
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("开始执行：" + method.getName());
        Object result = method.invoke(target, args);
        System.out.println("执行结束：" + method.getName());
        return result;
    }
}

// 使用示例
public class ProxyExample {
    public static void main(String[] args) {
        UserService userService = new UserServiceImpl();
        UserService proxy = (UserService) Proxy.newProxyInstance(
            userService.getClass().getClassLoader(),
            userService.getClass().getInterfaces(),
            new LogHandler(userService));
        
        proxy.addUser("张三");
        proxy.deleteUser("李四");
    }
}
```

### 3.3 CGLIB动态代理

```java
public class CGLIBProxy implements MethodInterceptor {
    private Object target;
    
    public Object getInstance(Object target) {
        this.target = target;
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(target.getClass());
        enhancer.setCallback(this);
        return enhancer.create();
    }
    
    @Override
    public Object intercept(Object obj, Method method, Object[] args,
            MethodProxy proxy) throws Throwable {
        System.out.println("开始执行：" + method.getName());
        Object result = proxy.invokeSuper(obj, args);
        System.out.println("执行结束：" + method.getName());
        return result;
    }
}
```

## 4. 实践案例

### 4.1 简单ORM框架

```java
@Table("users")
public class User {
    @Column("id")
    private Long id;
    
    @Column("name")
    private String name;
    
    // getters and setters
}

public class SimpleORM {
    public static String generateSelect(Class<?> clazz) {
        StringBuilder sql = new StringBuilder("SELECT ");
        
        // 获取表名
        Table table = clazz.getAnnotation(Table.class);
        String tableName = table.value();
        
        // 获取所有字段
        Field[] fields = clazz.getDeclaredFields();
        for (int i = 0; i < fields.length; i++) {
            Column column = fields[i].getAnnotation(Column.class);
            if (column != null) {
                if (i > 0) {
                    sql.append(", ");
                }
                sql.append(column.value());
            }
        }
        
        sql.append(" FROM ").append(tableName);
        return sql.toString();
    }
}
```

### 4.2 性能监控AOP

```java
public class PerformanceProxy implements InvocationHandler {
    private final Object target;
    
    public PerformanceProxy(Object target) {
        this.target = target;
    }
    
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        long start = System.currentTimeMillis();
        Object result = method.invoke(target, args);
        long end = System.currentTimeMillis();
        
        System.out.printf("方法 %s 执行时间：%d ms%n", 
            method.getName(), (end - start));
        return result;
    }
}
```

## 5. 最佳实践

1. 合理使用反射，避免滥用
2. 注意反射的性能开销
3. 缓存反射获取的类信息
4. 正确处理反射异常
5. 使用动态代理实现横切关注点

## 总结

本文介绍了Java反射和动态代理的核心概念和实践应用，包括：

1. 反射的基本概念和使用方法
2. 动态代理的实现方式
3. JDK代理和CGLIB代理的区别
4. 实际应用案例
5. 最佳实践建议

通过掌握这些知识，我们可以更好地理解框架的实现原理，并在实际开发中灵活运用这些特性。

## 参考资源

1. Java官方文档：https://docs.oracle.com/javase/tutorial/reflect/
2. Spring框架中的反射应用
3. CGLIB官方文档
4. Java反射和动态代理最佳实践指南