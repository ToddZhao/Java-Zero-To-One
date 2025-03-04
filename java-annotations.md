# Day 7: Java注解和元数据

## 引言

Java注解（Annotations）是一种元数据形式，它为我们提供了一种在代码中添加信息的方式，这些信息可以在编译时、类加载时或运行时被处理。本文将深入介绍Java注解的核心概念和实践应用。

## 1. 注解基础

### 1.1 什么是注解

注解是一种特殊的接口，它可以声明在包、类、方法、字段等上面，为它们提供额外的信息：

- 编译器信息：帮助编译器检测错误或抑制警告
- 编译时和部署时处理：生成代码、XML文件等
- 运行时处理：运行时可以通过反射获取注解信息

### 1.2 内置注解

Java提供了一些内置的注解：

```java
// 标记过时的元素
@Deprecated
public void oldMethod() {}

// 抑制警告
@SuppressWarnings("unchecked")
public void suppressMethod() {}

// 重写方法
@Override
public String toString() {
    return "Example";
}

// 函数式接口
@FunctionalInterface
public interface MyFunction {
    void execute();
}
```

## 2. 自定义注解

### 2.1 创建注解

```java
// 定义注解
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface MyAnnotation {
    String value() default "";
    int count() default 0;
    String[] tags() default {};
}

// 使用注解
public class MyClass {
    @MyAnnotation(value = "测试", count = 1, tags = {"tag1", "tag2"})
    public void testMethod() {
        // 方法实现
    }
}
```

### 2.2 注解处理器

```java
public class AnnotationProcessor {
    public static void processAnnotations(Class<?> clazz) {
        // 获取所有方法
        for (Method method : clazz.getDeclaredMethods()) {
            // 检查方法是否有指定注解
            if (method.isAnnotationPresent(MyAnnotation.class)) {
                // 获取注解实例
                MyAnnotation annotation = method.getAnnotation(MyAnnotation.class);
                
                // 处理注解信息
                System.out.println("方法：" + method.getName());
                System.out.println("注解值：" + annotation.value());
                System.out.println("计数：" + annotation.count());
                System.out.println("标签：" + Arrays.toString(annotation.tags()));
            }
        }
    }
}
```

## 3. 元注解

元注解用于注解其他注解：

```java
// @Target：指定注解可以应用的元素类型
@Target({ElementType.METHOD, ElementType.FIELD})

// @Retention：指定注解的保留策略
@Retention(RetentionPolicy.RUNTIME)

// @Documented：指定注解应该包含在JavaDoc中
@Documented

// @Inherited：指定注解可以被继承
@Inherited

// 使用元注解创建自定义注解
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface MethodInfo {
    String author() default "unknown";
    String date();
    int revision() default 1;
    String comments();
}
```

## 4. 实践案例

### 4.1 参数校验注解

```java
// 校验注解
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
public @interface ValidateField {
    int minLength() default 0;
    int maxLength() default Integer.MAX_VALUE;
    String pattern() default "";
    boolean required() default true;
}

// 使用注解的实体类
public class User {
    @ValidateField(minLength = 2, maxLength = 50)
    private String name;
    
    @ValidateField(pattern = "^[\\w-\\.]+@([\\w-]+\\.)+[\\w-]{2,4}$")
    private String email;
    
    // getters and setters
}

// 验证处理器
public class ValidationProcessor {
    public static List<String> validate(Object obj) {
        List<String> errors = new ArrayList<>();
        Class<?> clazz = obj.getClass();
        
        for (Field field : clazz.getDeclaredFields()) {
            if (field.isAnnotationPresent(ValidateField.class)) {
                ValidateField annotation = field.getAnnotation(ValidateField.class);
                field.setAccessible(true);
                
                try {
                    Object value = field.get(obj);
                    if (value instanceof String) {
                        String strValue = (String) value;
                        
                        // 必填校验
                        if (annotation.required() && 
                            (strValue == null || strValue.trim().isEmpty())) {
                            errors.add(field.getName() + "不能为空");
                            continue;
                        }
                        
                        // 长度校验
                        if (strValue != null) {
                            if (strValue.length() < annotation.minLength()) {
                                errors.add(field.getName() + "长度不能小于" + 
                                    annotation.minLength());
                            }
                            if (strValue.length() > annotation.maxLength()) {
                                errors.add(field.getName() + "长度不能大于" + 
                                    annotation.maxLength());
                            }
                        }
                        
                        // 正则校验
                        if (!annotation.pattern().isEmpty() && 
                            !strValue.matches(annotation.pattern())) {
                            errors.add(field.getName() + "格式不正确");
                        }
                    }
                } catch (IllegalAccessException e) {
                    e.printStackTrace();
                }
            }
        }
        
        return errors;
    }
}
```

### 4.2 性能监控注解

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface LogExecutionTime {
    String value() default "";
}

public class PerformanceAspect {
    public static void logExecutionTime(ProceedingJoinPoint joinPoint) throws Throwable {
        Method method = ((MethodSignature) joinPoint.getSignature()).getMethod();
        LogExecutionTime annotation = method.getAnnotation(LogExecutionTime.class);
        
        long start = System.currentTimeMillis();
        Object result = joinPoint.proceed();
        long end = System.currentTimeMillis();
        
        String methodName = annotation.value().isEmpty() ? 
            method.getName() : annotation.value();
        System.out.printf("%s 执行时间：%d ms%n", methodName, (end - start));
    }
}
```

## 5. 最佳实践

1. 合理使用注解，避免过度使用
2. 注解处理器要高效且线程安全
3. 注解命名要清晰明确
4. 提供适当的默认值
5. 注意注解的保留策略

## 总结

本文介绍了Java注解和元数据的核心概念和实践应用，包括：

1. 注解的基本概念和使用
2. 自定义注解的创建和处理
3. 元注解的应用
4. 实践案例
5. 最佳实践建议

通过掌握这些知识，我们可以更好地利用注解来提高代码的可读性和可维护性，实现更优雅的程序设计。

## 参考资源

1. Java官方文档：https://docs.oracle.com/javase/tutorial/java/annotations/
2. Spring框架中的注解应用
3. Java注解处理器指南
4. 注解最佳实践指南