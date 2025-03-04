# Day 18: Java Spring框架 - AOP

## 引言

AOP（面向切面编程）是Spring框架的重要特性之一，它通过预编译和运行期动态代理实现程序功能的统一维护。本文将详细介绍Spring AOP的核心概念和实践应用。

## 1. AOP基础

### 1.1 AOP术语

- 切面（Aspect）：横切关注点的模块化
- 连接点（Join Point）：程序执行的某个特定位置
- 通知（Advice）：在切面的某个特定连接点上执行的动作
- 切入点（Pointcut）：匹配连接点的表达式
- 目标对象（Target Object）：被代理的对象

### 1.2 通知类型

```java
@Aspect
@Component
public class LoggingAspect {
    // 前置通知
    @Before("execution(* com.example.service.*.*(..))"))
    public void beforeMethod(JoinPoint joinPoint) {
        String methodName = joinPoint.getSignature().getName();
        System.out.println("开始执行方法：" + methodName);
    }
    
    // 后置通知
    @After("execution(* com.example.service.*.*(..))"))
    public void afterMethod(JoinPoint joinPoint) {
        String methodName = joinPoint.getSignature().getName();
        System.out.println("方法执行结束：" + methodName);
    }
    
    // 返回通知
    @AfterReturning(value = "execution(* com.example.service.*.*(..))", 
                    returning = "result")
    public void afterReturning(JoinPoint joinPoint, Object result) {
        System.out.println("方法返回值：" + result);
    }
    
    // 异常通知
    @AfterThrowing(value = "execution(* com.example.service.*.*(..))", 
                   throwing = "ex")
    public void afterThrowing(JoinPoint joinPoint, Exception ex) {
        System.out.println("方法抛出异常：" + ex.getMessage());
    }
    
    // 环绕通知
    @Around("execution(* com.example.service.*.*(..))"))
    public Object around(ProceedingJoinPoint joinPoint) throws Throwable {
        System.out.println("方法执行前");
        Object result = joinPoint.proceed();
        System.out.println("方法执行后");
        return result;
    }
}
```

## 2. 切入点表达式

### 2.1 常用表达式

```java
@Aspect
@Component
public class AspectDemo {
    // 匹配任何public方法
    @Pointcut("execution(public * *(..))"))
    public void publicMethod() {}
    
    // 匹配特定包下的任何方法
    @Pointcut("execution(* com.example.service.*.*(..))"))
    public void serviceLayer() {}
    
    // 匹配带有特定注解的方法
    @Pointcut("@annotation(com.example.annotation.LogExecutionTime)")
    public void logExecutionTime() {}
    
    // 组合切入点
    @Pointcut("publicMethod() && serviceLayer()")
    public void publicServiceMethod() {}
}
```

### 2.2 参数绑定

```java
@Aspect
@Component
public class ParameterBindingAspect {
    @Before("execution(* com.example.service.*.*(..)) && args(id,..)"))
    public void beforeMethodWithId(Long id) {
        System.out.println("处理ID：" + id);
    }
    
    @Before("@annotation(logExecutionTime)")
    public void beforeMethodWithAnnotation(LogExecutionTime logExecutionTime) {
        System.out.println("注解值：" + logExecutionTime.value());
    }
}
```

## 3. 实现方式

### 3.1 基于注解

```java
// 自定义注解
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface LogExecutionTime {
    String value() default "";
}

// 使用注解
@Service
public class UserService {
    @LogExecutionTime("查询用户")
    public User findById(Long id) {
        // 方法实现
    }
}

// 切面实现
@Aspect
@Component
public class LogExecutionTimeAspect {
    @Around("@annotation(logExecutionTime)")
    public Object logExecutionTime(ProceedingJoinPoint joinPoint, 
            LogExecutionTime logExecutionTime) throws Throwable {
        long start = System.currentTimeMillis();
        Object result = joinPoint.proceed();
        long end = System.currentTimeMillis();
        
        System.out.printf("%s 执行时间：%d ms%n", 
            logExecutionTime.value(), (end - start));
        return result;
    }
}
```

### 3.2 基于XML配置

```xml
<aop:config>
    <aop:aspect id="loggingAspect" ref="loggingBean">
        <aop:pointcut id="serviceMethod"
            expression="execution(* com.example.service.*.*(..))" />
        
        <aop:before pointcut-ref="serviceMethod" 
            method="beforeMethod"/>
        <aop:after pointcut-ref="serviceMethod" 
            method="afterMethod"/>
        <aop:around pointcut-ref="serviceMethod" 
            method="aroundMethod"/>
    </aop:aspect>
</aop:config>
```

## 4. 实践案例

### 4.1 性能监控

```java
@Aspect
@Component
public class PerformanceMonitorAspect {
    private final Map<String, MethodStats> methodStats = new ConcurrentHashMap<>();
    
    @Around("execution(* com.example.service.*.*(..))")
    public Object monitorPerformance(ProceedingJoinPoint joinPoint) 
            throws Throwable {
        String methodName = joinPoint.getSignature().toShortString();
        MethodStats stats = methodStats.computeIfAbsent(methodName, 
            k -> new MethodStats());
        
        long start = System.nanoTime();
        try {
            return joinPoint.proceed();
        } finally {
            long end = System.nanoTime();
            stats.addExecutionTime(end - start);
        }
    }
    
    @Scheduled(fixedRate = 60000)
    public void logStats() {
        methodStats.forEach((method, stats) -> {
            System.out.printf("%s - 平均执行时间：%.2f ms, 调用次数：%d%n",
                method, stats.getAverageTime() / 1_000_000.0, 
                stats.getCallCount());
        });
    }
}

class MethodStats {
    private final AtomicLong totalTime = new AtomicLong();
    private final AtomicInteger callCount = new AtomicInteger();
    
    public void addExecutionTime(long time) {
        totalTime.addAndGet(time);
        callCount.incrementAndGet();
    }
    
    public double getAverageTime() {
        return callCount.get() > 0 ? 
            (double) totalTime.get() / callCount.get() : 0;
    }
    
    public int getCallCount() {
        return callCount.get();
    }
}
```

### 4.2 安全检查

```java
@Aspect
@Component
public class SecurityAspect {
    @Autowired
    private SecurityService securityService;
    
    @Before("@annotation(requiresPermission)")
    public void checkPermission(JoinPoint joinPoint, 
            RequiresPermission requiresPermission) {
        String permission = requiresPermission.value();
        if (!securityService.hasPermission(permission)) {
            throw new AccessDeniedException(
                "没有权限执行此操作：" + permission);
        }
    }
}

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface RequiresPermission {
    String value();
}

@Service
public class UserService {
    @RequiresPermission("user:delete")
    public void deleteUser(Long id) {
        // 删除用户
    }
}
```

## 5. 最佳实践

1. 合理使用AOP，避免过度使用
2. 选择合适的切入点表达式
3. 注意切面的优先级
4. 处理好异常情况
5. 考虑性能影响

## 总结

本文介绍了Spring AOP的核心概念和实践应用，包括：

1. AOP的基本概念
2. 切入点表达式的使用
3. 不同类型的通知
4. 基于注解和XML的配置
5. 实践案例和最佳实践

通过掌握这些知识，我们可以更好地使用AOP来处理横切关注点，提高代码的模块化程度。

## 参考资源

1. Spring官方文档：https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#aop
2. Spring AOP实战
3. AspectJ编程指南
4. Spring AOP最佳实践