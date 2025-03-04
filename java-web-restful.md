# Day 27: Java Web开发 - RESTful API

## 引言

RESTful API是一种基于REST架构风格的应用程序接口，它使用HTTP协议的标准方法来进行资源的访问和操作。本文将详细介绍RESTful API的设计原则和实践应用。

## 1. REST基础

### 1.1 REST架构约束

- 客户端-服务器：分离客户端和服务器
- 无状态：每个请求包含所有必要信息
- 可缓存：响应可以被缓存
- 统一接口：使用标准的HTTP方法
- 分层系统：允许中间层的存在

### 1.2 HTTP方法

```java
@RestController
@RequestMapping("/api/users")
public class UserController {
    @Autowired
    private UserService userService;
    
    // GET：获取资源
    @GetMapping
    public List<User> getUsers() {
        return userService.findAll();
    }
    
    // POST：创建资源
    @PostMapping
    public ResponseEntity<User> createUser(@RequestBody User user) {
        User savedUser = userService.save(user);
        return ResponseEntity
            .created(URI.create("/api/users/" + savedUser.getId()))
            .body(savedUser);
    }
    
    // PUT：更新资源
    @PutMapping("/{id}")
    public ResponseEntity<User> updateUser(
            @PathVariable Long id, 
            @RequestBody User user) {
        user.setId(id);
        User updatedUser = userService.update(user);
        return ResponseEntity.ok(updatedUser);
    }
    
    // DELETE：删除资源
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteUser(@PathVariable Long id) {
        userService.delete(id);
        return ResponseEntity.noContent().build();
    }
}
```

## 2. 资源设计

### 2.1 资源命名

```java
// 使用名词复数形式
@RestController
@RequestMapping("/api/products")
public class ProductController {
    
    // 获取产品列表
    @GetMapping
    public List<Product> getProducts() { ... }
    
    // 获取单个产品
    @GetMapping("/{id}")
    public Product getProduct(@PathVariable Long id) { ... }
    
    // 获取产品的评论
    @GetMapping("/{id}/reviews")
    public List<Review> getProductReviews(@PathVariable Long id) { ... }
}
```

### 2.2 查询参数

```java
@RestController
@RequestMapping("/api/products")
public class ProductController {
    
    @GetMapping
    public Page<Product> getProducts(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "10") int size,
            @RequestParam(required = false) String category,
            @RequestParam(required = false) BigDecimal minPrice) {
        
        return productService.findProducts(page, size, category, minPrice);
    }
}
```

## 3. 响应设计

### 3.1 状态码使用

```java
@RestController
@RequestMapping("/api/orders")
public class OrderController {
    
    // 200 OK：成功获取资源
    @GetMapping("/{id}")
    public ResponseEntity<Order> getOrder(@PathVariable Long id) {
        return orderService.findById(id)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }
    
    // 201 Created：成功创建资源
    @PostMapping
    public ResponseEntity<Order> createOrder(@RequestBody Order order) {
        Order savedOrder = orderService.save(order);
        return ResponseEntity
            .created(URI.create("/api/orders/" + savedOrder.getId()))
            .body(savedOrder);
    }
    
    // 204 No Content：成功处理但无返回内容
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteOrder(@PathVariable Long id) {
        orderService.delete(id);
        return ResponseEntity.noContent().build();
    }
}
```

### 3.2 错误处理

```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    
    // 404 Not Found
    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(ResourceNotFoundException ex) {
        ErrorResponse error = new ErrorResponse(
            HttpStatus.NOT_FOUND.value(),
            ex.getMessage());
        return new ResponseEntity<>(error, HttpStatus.NOT_FOUND);
    }
    
    // 400 Bad Request
    @ExceptionHandler(ValidationException.class)
    public ResponseEntity<ErrorResponse> handleValidation(ValidationException ex) {
        ErrorResponse error = new ErrorResponse(
            HttpStatus.BAD_REQUEST.value(),
            ex.getMessage());
        return new ResponseEntity<>(error, HttpStatus.BAD_REQUEST);
    }
    
    // 500 Internal Server Error
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleInternal(Exception ex) {
        ErrorResponse error = new ErrorResponse(
            HttpStatus.INTERNAL_SERVER_ERROR.value(),
            "Internal server error");
        return new ResponseEntity<>(error, HttpStatus.INTERNAL_SERVER_ERROR);
    }
}

@Data
@AllArgsConstructor
public class ErrorResponse {
    private int status;
    private String message;
    private List<String> errors;
}
```

## 4. 版本控制

### 4.1 URL版本控制

```java
@RestController
@RequestMapping("/api/v1/users")
public class UserControllerV1 {
    // V1版本API实现
}

@RestController
@RequestMapping("/api/v2/users")
public class UserControllerV2 {
    // V2版本API实现
}
```

### 4.2 请求头版本控制

```java
@RestController
@RequestMapping("/api/users")
public class UserController {
    
    @GetMapping(headers = "API-Version=1")
    public ResponseEntity<UserV1> getUserV1(@PathVariable Long id) {
        // V1版本实现
    }
    
    @GetMapping(headers = "API-Version=2")
    public ResponseEntity<UserV2> getUserV2(@PathVariable Long id) {
        // V2版本实现
    }
}
```

## 5. 安全性

### 5.1 认证和授权

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.csrf().disable()
            .authorizeRequests()
            .antMatchers("/api/public/**").permitAll()
            .antMatchers("/api/admin/**").hasRole("ADMIN")
            .anyRequest().authenticated()
            .and()
            .addFilter(new JwtAuthenticationFilter())
            .addFilter(new JwtAuthorizationFilter());
    }
}

@RestController
@RequestMapping("/api/secured")
public class SecuredController {
    
    @GetMapping("/resource")
    @PreAuthorize("hasRole('USER')")
    public ResponseEntity<Resource> getSecuredResource() {
        // 获取受保护的资源
    }
}
```

## 6. 实践案例

### 6.1 分页和排序

```java
@RestController
@RequestMapping("/api/products")
public class ProductController {
    
    @GetMapping
    public ResponseEntity<Page<Product>> getProducts(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "10") int size,
            @RequestParam(defaultValue = "name") String sortBy,
            @RequestParam(defaultValue = "asc") String direction) {
        
        Sort.Direction sortDirection = Sort.Direction.fromString(direction);
        Pageable pageable = PageRequest.of(page, size, Sort.by(sortDirection, sortBy));
        
        Page<Product> products = productService.findAll(pageable);
        return ResponseEntity.ok(products);
    }
}
```

### 6.2 条件过滤

```java
@RestController
@RequestMapping("/api/products")
public class ProductController {
    
    @GetMapping("/search")
    public ResponseEntity<List<Product>> searchProducts(
            @RequestParam(required = false) String category,
            @RequestParam(required = false) BigDecimal minPrice,
            @RequestParam(required = false) BigDecimal maxPrice,
            @RequestParam(required = false) List<String> tags) {
        
        ProductSearchCriteria criteria = ProductSearchCriteria.builder()
            .category(category)
            .minPrice(minPrice)
            .maxPrice(maxPrice)
            .tags(tags)
            .build();
        
        List<Product> products = productService.search(criteria);
        return ResponseEntity.ok(products);
    }
}
```

## 7. 最佳实践

1. 使用正确的HTTP方法
2. 提供清晰的错误信息
3. 实现适当的版本控制
4. 使用HTTPS保护API
5. 实现请求限流

## 总结

本文介绍了RESTful API的核心概念和实践应用，包括：

1. REST架构约束
2. 资源设计原则
3. 响应处理
4. 版本控制策略
5. 安全性考虑
6. 实践案例

通过掌握这些知识，我们可以设计和实现出符合REST规范的、易于使用和维护的Web API。

## 参考资源

1. RESTful Web Services规范
2. Spring REST文档
3. Web API设计最佳实践
4. REST API安全指南