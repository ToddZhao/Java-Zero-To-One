# Day 33: Java单元测试 - JUnit和Mockito

## 引言

单元测试是确保代码质量的重要手段。在Java生态系统中，JUnit和Mockito是两个最流行的测试框架。JUnit提供了基础的测试框架，而Mockito则提供了强大的模拟功能。本文将详细介绍这两个框架的使用方法和最佳实践。

## 1. JUnit基础

### 1.1 JUnit特点

- 简单易用
- 注解驱动
- 断言机制
- 测试生命周期管理
- 测试套件支持

### 1.2 基本注解

```java
import org.junit.jupiter.api.*;
import static org.junit.jupiter.api.Assertions.*;

public class CalculatorTest {

    private Calculator calculator;

    @BeforeEach
    void setUp() {
        calculator = new Calculator();
    }

    @Test
    void testAdd() {
        assertEquals(4, calculator.add(2, 2));
    }

    @Test
    void testDivide() {
        assertThrows(ArithmeticException.class, () -> 
            calculator.divide(1, 0));
    }

    @ParameterizedTest
    @ValueSource(ints = {1, 2, 3})
    void testMultiply(int value) {
        assertEquals(value * 2, calculator.multiply(value, 2));
    }

    @AfterEach
    void tearDown() {
        calculator = null;
    }
}
```

### 1.3 断言方法

```java
@Test
void testAssertions() {
    // 相等性测试
    assertEquals(expected, actual);
    assertNotEquals(unexpected, actual);

    // 布尔测试
    assertTrue(condition);
    assertFalse(condition);

    // 空值测试
    assertNull(object);
    assertNotNull(object);

    // 引用测试
    assertSame(expected, actual);
    assertNotSame(unexpected, actual);

    // 数组测试
    assertArrayEquals(expectedArray, actualArray);

    // 异常测试
    assertThrows(ExpectedException.class, () -> {
        // 可能抛出异常的代码
    });
}
```

## 2. Mockito基础

### 2.1 Mockito特点

- 简洁的API
- 验证方法调用
- 灵活的参数匹配
- 支持spy对象
- 注解支持

### 2.2 基本用法

```java
import static org.mockito.Mockito.*;
import org.mockito.Mock;
import org.mockito.InjectMocks;

@ExtendWith(MockitoExtension.class)
public class UserServiceTest {

    @Mock
    private UserRepository userRepository;

    @InjectMocks
    private UserService userService;

    @Test
    void testGetUser() {
        // 设置模拟行为
        when(userRepository.findById(1L))
            .thenReturn(new User(1L, "张三"));

        // 调用被测试方法
        User user = userService.getUser(1L);

        // 验证结果
        assertEquals("张三", user.getName());
        verify(userRepository).findById(1L);
    }

    @Test
    void testSaveUser() {
        User user = new User(1L, "张三");
        
        // 设置模拟行为
        when(userRepository.save(any(User.class)))
            .thenReturn(user);

        // 调用被测试方法
        userService.saveUser(user);

        // 验证方法调用
        verify(userRepository).save(user);
    }
}
```

### 2.3 参数匹配器

```java
@Test
void testArgumentMatchers() {
    // 任意参数
    when(service.process(any())).thenReturn(result);

    // 特定类型
    when(service.process(anyString())).thenReturn(result);

    // 自定义匹配
    when(service.process(argThat(str -> str.length() > 5)))
        .thenReturn(result);

    // 多个参数
    when(service.process(anyString(), anyInt()))
        .thenReturn(result);
}
```

## 3. 实践案例

### 3.1 业务逻辑测试

```java
public class OrderServiceTest {

    @Mock
    private OrderRepository orderRepository;
    @Mock
    private PaymentService paymentService;
    @Mock
    private NotificationService notificationService;

    @InjectMocks
    private OrderService orderService;

    @Test
    void testCreateOrder() {
        // 准备测试数据
        Order order = new Order();
        order.setAmount(100.0);

        // 设置模拟行为
        when(orderRepository.save(any(Order.class)))
            .thenReturn(order);
        when(paymentService.process(anyDouble()))
            .thenReturn(true);

        // 执行测试
        Order result = orderService.createOrder(order);

        // 验证结果
        assertNotNull(result);
        assertEquals(100.0, result.getAmount());

        // 验证交互
        verify(orderRepository).save(order);
        verify(paymentService).process(100.0);
        verify(notificationService).sendConfirmation(order);
    }
}
```

### 3.2 异常处理测试

```java
public class ExceptionHandlingTest {

    @Test
    void testExceptionHandling() {
        // 设置模拟行为抛出异常
        when(service.riskyOperation())
            .thenThrow(new RuntimeException("Error"));

        // 验证异常处理
        assertThrows(ServiceException.class, () -> {
            service.executeWithRetry();
        });

        // 验证重试逻辑
        verify(service, times(3)).riskyOperation();
    }
}
```

## 4. 测试最佳实践

1. 测试命名规范
```java
@Test
void shouldReturnUserWhenUserExists() {
    // 测试代码
}

@Test
void shouldThrowExceptionWhenUserNotFound() {
    // 测试代码
}
```

2. 测试隔离
```java
@Test
void testIsolation() {
    // 每个测试都应该是独立的
    // 不依赖其他测试的状态
    // 不依赖外部资源
}
```

3. 测试覆盖率
```java
// 使用JaCoCo等工具监控测试覆盖率
// 关注核心业务逻辑的测试覆盖
// 不过分追求100%覆盖率
```

4. 可维护性
- 遵循DRY原则
- 使用测试工具类
- 适当的注释

## 5. 高级特性

1. 测试生命周期
```java
@TestInstance(TestInstance.Lifecycle.PER_CLASS)
class LifecycleTest {

    @BeforeAll
    void initAll() {
        // 所有测试之前执行
    }

    @BeforeEach
    void init() {
        // 每个测试之前执行
    }

    @AfterEach
    void tearDown() {
        // 每个测试之后执行
    }

    @AfterAll
    void tearDownAll() {
        // 所有测试之后执行
    }
}
```

2. 参数化测试
```java
@ParameterizedTest
@CsvSource({
    "1, 2, 3",
    "5, 3, 8",
    "10, -2, 8"
})
void testAdd(int a, int b, int expected) {
    assertEquals(expected, calculator.add(a, b));
}
```

## 总结

本文介绍了Java单元测试的两个主要框架：JUnit和Mockito，包括：

1. JUnit的基本用法和特性
2. Mockito的模拟功能和验证机制
3. 实践案例
4. 测试最佳实践
5. 高级特性应用

掌握这些测试框架和最佳实践，可以帮助我们编写更可靠的代码，提高代码质量。

## 参考资源

1. JUnit 5官方文档：https://junit.org/junit5/docs/current/user-guide/
2. Mockito官方文档：https://javadoc.io/doc/org.mockito/mockito-core/latest/org/mockito/Mockito.html
3. Java测试最佳实践指南
4. 测试驱动开发实践指南