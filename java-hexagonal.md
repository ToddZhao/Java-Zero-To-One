# Day 123: Java架构设计 - 六边形架构

## 1. 六边形架构概述

六边形架构（Hexagonal Architecture），也称为端口与适配器架构（Ports and Adapters Architecture），是一种将应用核心逻辑与外部系统交互分离的软件架构模式。这种架构模式由Alistair Cockburn于2005年提出，旨在创建松耦合、可测试且可维护的系统。

## 2. 六边形架构核心概念

### 2.1 领域层（Domain）

```java
// 领域实体
public class Product {
    private ProductId id;
    private String name;
    private String description;
    private Money price;
    private int stockQuantity;
    
    public Product(ProductId id, String name, String description, Money price, int stockQuantity) {
        this.id = id;
        this.name = name;
        this.description = description;
        this.price = price;
        this.stockQuantity = stockQuantity;
    }
    
    public void decreaseStock(int quantity) {
        if (quantity <= 0) {
            throw new IllegalArgumentException("Quantity must be positive");
        }
        
        if (quantity > stockQuantity) {
            throw new InsufficientStockException(id, stockQuantity, quantity);
        }
        
        this.stockQuantity -= quantity;
    }
    
    public void increaseStock(int quantity) {
        if (quantity <= 0) {
            throw new IllegalArgumentException("Quantity must be positive");
        }
        
        this.stockQuantity += quantity;
    }
    
    // getters
}

// 领域服务
public class OrderService {
    private final ProductRepository productRepository;
    private final OrderRepository orderRepository;
    
    public OrderService(ProductRepository productRepository, OrderRepository orderRepository) {
        this.productRepository = productRepository;
        this.orderRepository = orderRepository;
    }
    
    public Order createOrder(CustomerId customerId, List<OrderItem> items) {
        // 验证库存
        for (OrderItem item : items) {
            Product product = productRepository.findById(item.getProductId())
                .orElseThrow(() -> new ProductNotFoundException(item.getProductId()));
                
            product.decreaseStock(item.getQuantity());
            productRepository.save(product);
        }
        
        // 创建订单
        Order order = new Order(OrderId.generate(), customerId, items, OrderStatus.CREATED);
        orderRepository.save(order);
        
        return order;
    }
}
```

### 2.2 端口（Ports）

```java
// 入站端口（Primary/Driving Ports）
public interface OrderApplicationService {
    OrderDTO createOrder(CreateOrderCommand command);
    OrderDTO getOrder(OrderId orderId);
    void cancelOrder(OrderId orderId);
}

// 出站端口（Secondary/Driven Ports）
public interface ProductRepository {
    Optional<Product> findById(ProductId id);
    void save(Product product);
}

public interface OrderRepository {
    Optional<Order> findById(OrderId id);
    void save(Order order);
}

public interface PaymentGateway {
    PaymentResult processPayment(OrderId orderId, Money amount, PaymentDetails details);
}
```

### 2.3 适配器（Adapters）

```java
// 入站适配器（Primary/Driving Adapters）
@RestController
@RequestMapping("/api/orders")
public class OrderController {
    private final OrderApplicationService orderApplicationService;
    
    public OrderController(OrderApplicationService orderApplicationService) {
        this.orderApplicationService = orderApplicationService;
    }
    
    @PostMapping
    public ResponseEntity<OrderDTO> createOrder(@RequestBody CreateOrderRequest request) {
        CreateOrderCommand command = new CreateOrderCommand(
            request.getCustomerId(),
            request.getItems().stream()
                .map(item -> new OrderItemCommand(item.getProductId(), item.getQuantity()))
                .collect(Collectors.toList())
        );
        
        OrderDTO order = orderApplicationService.createOrder(command);
        return ResponseEntity.ok(order);
    }
    
    @GetMapping("/{orderId}")
    public ResponseEntity<OrderDTO> getOrder(@PathVariable String orderId) {
        OrderDTO order = orderApplicationService.getOrder(new OrderId(orderId));
        return ResponseEntity.ok(order);
    }
    
    @DeleteMapping("/{orderId}")
    public ResponseEntity<Void> cancelOrder(@PathVariable String orderId) {
        orderApplicationService.cancelOrder(new OrderId(orderId));
        return ResponseEntity.noContent().build();
    }
}

// 出站适配器（Secondary/Driven Adapters）
@Repository
public class JpaProductRepository implements ProductRepository {
    private final ProductJpaRepository jpaRepository;
    private final ProductMapper mapper;
    
    public JpaProductRepository(ProductJpaRepository jpaRepository, ProductMapper mapper) {
        this.jpaRepository = jpaRepository;
        this.mapper = mapper;
    }
    
    @Override
    public Optional<Product> findById(ProductId id) {
        return jpaRepository.findById(id.getValue())
            .map(mapper::toDomain);
    }
    
    @Override
    public void save(Product product) {
        ProductEntity entity = mapper.toEntity(product);
        jpaRepository.save(entity);
    }
}

@Service
public class StripePaymentAdapter implements PaymentGateway {
    private final StripeClient stripeClient;
    
    public StripePaymentAdapter(StripeClient stripeClient) {
        this.stripeClient = stripeClient;
    }
    
    @Override
    public PaymentResult processPayment(OrderId orderId, Money amount, PaymentDetails details) {
        try {
            StripePaymentRequest request = new StripePaymentRequest(
                details.getCardNumber(),
                details.getExpiryMonth(),
                details.getExpiryYear(),
                details.getCvv(),
                amount.getAmount(),
                amount.getCurrency()
            );
            
            StripePaymentResponse response = stripeClient.processPayment(request);
            
            return new PaymentResult(
                response.isSuccessful(),
                response.getTransactionId(),
                response.getMessage()
            );
        } catch (StripeException e) {
            return new PaymentResult(false, null, e.getMessage());
        }
    }
}
```

## 3. 应用层实现

```java
@Service
public class OrderApplicationServiceImpl implements OrderApplicationService {
    private final OrderService orderService;
    private final ProductRepository productRepository;
    private final OrderRepository orderRepository;
    private final PaymentGateway paymentGateway;
    private final OrderMapper mapper;
    
    public OrderApplicationServiceImpl(
            OrderService orderService,
            ProductRepository productRepository,
            OrderRepository orderRepository,
            PaymentGateway paymentGateway,
            OrderMapper mapper) {
        this.orderService = orderService;
        this.productRepository = productRepository;
        this.orderRepository = orderRepository;
        this.paymentGateway = paymentGateway;
        this.mapper = mapper;
    }
    
    @Override
    @Transactional
    public OrderDTO createOrder(CreateOrderCommand command) {
        List<OrderItem> items = command.getItems().stream()
            .map(item -> new OrderItem(item.getProductId(), item.getQuantity()))
            .collect(Collectors.toList());
        
        Order order = orderService.createOrder(command.getCustomerId(), items);
        return mapper.toDTO(order);
    }
    
    @Override
    @Transactional(readOnly = true)
    public OrderDTO getOrder(OrderId orderId) {
        Order order = orderRepository.findById(orderId)
            .orElseThrow(() -> new OrderNotFoundException(orderId));
        return mapper.toDTO(order);
    }
    
    @Override
    @Transactional
    public void cancelOrder(OrderId orderId) {
        Order order = orderRepository.findById(orderId)
            .orElseThrow(() -> new OrderNotFoundException(orderId));
        
        order.cancel();
        orderRepository.save(order);
        
        // 恢复库存
        for (OrderItem item : order.getItems()) {
            Product product = productRepository.findById(item.getProductId())
                .orElseThrow(() -> new ProductNotFoundException(item.getProductId()));
            
            product.increaseStock(item.getQuantity());
            productRepository.save(product);
        }
    }
}
```

## 4. 配置与依赖注入

```java
@Configuration
public class ApplicationConfig {
    
    @Bean
    public OrderService orderService(ProductRepository productRepository, OrderRepository orderRepository) {
        return new OrderService(productRepository, orderRepository);
    }
    
    @Bean
    public OrderApplicationService orderApplicationService(
            OrderService orderService,
            ProductRepository productRepository,
            OrderRepository orderRepository,
            PaymentGateway paymentGateway,
            OrderMapper mapper) {
        return new OrderApplicationServiceImpl(
            orderService,
            productRepository,
            orderRepository,
            paymentGateway,
            mapper
        );
    }
    
    @Bean
    public PaymentGateway paymentGateway(StripeClient stripeClient) {
        return new StripePaymentAdapter(stripeClient);
    }
}
```

## 5. 测试策略

```java
public class OrderServiceTest {
    
    private OrderService orderService;
    private ProductRepository productRepository;
    private OrderRepository orderRepository;
    
    @BeforeEach
    public void setup() {
        productRepository = mock(ProductRepository.class);
        orderRepository = mock(OrderRepository.class);
        orderService = new OrderService(productRepository, orderRepository);
    }
    
    @Test
    public void createOrder_WithValidData_ShouldCreateOrderAndUpdateStock() {
        // Arrange
        CustomerId customerId = new CustomerId("customer-1");
        ProductId productId = new ProductId("product-1");
        int quantity = 2;
        
        Product product = new Product(
            productId,
            "Test Product",
            "Description",
            new Money(new BigDecimal("10.00"), "USD"),
            10
        );
        
        when(productRepository.findById(productId)).thenReturn(Optional.of(product));
        
        // Act
        List<OrderItem> items = Collections.singletonList(new OrderItem(productId, quantity));
        Order order = orderService.createOrder(customerId, items);
        
        // Assert
        assertEquals(customerId, order.getCustomerId());
        assertEquals(OrderStatus.CREATED, order.getStatus());
        assertEquals(1, order.getItems().size());
        
        verify(productRepository).findById(productId);
        verify(productRepository).save(product);
        verify(orderRepository).save(order);
        
        assertEquals(8, product.getStockQuantity()); // 10 - 2 = 8
    }
    
    @Test
    public void createOrder_WithInsufficientStock_ShouldThrowException() {
        // Arrange
        CustomerId customerId = new CustomerId("customer-1");
        ProductId productId = new ProductId("product-1");
        int quantity = 15; // 超过库存
        
        Product product = new Product(
            productId,
            "Test Product",
            "Description",
            new Money(new BigDecimal("10.00"), "USD"),
            10
        );
        
        when(productRepository.findById(productId)).thenReturn(Optional.of(product));
        
        // Act & Assert
        List<OrderItem> items = Collections.singletonList(new OrderItem(productId, quantity));
        assertThrows(InsufficientStockException.class, () -> {
            orderService.createOrder(customerId, items);
        });
    }
}
```

## 6. 最佳实践

### 6.1 依赖倒置原则

六边形架构的核心是依赖倒置原则（Dependency Inversion Principle），确保领域层不依赖于基础设施层：

```java
// 错误的依赖方向
public class OrderService {
    private final JpaOrderRepository orderRepository; // 直接依赖实现
    
    // ...
}

// 正确的依赖方向
public class OrderService {
    private final OrderRepository orderRepository; // 依赖接口
    
    // ...
}
```

### 6.2 领域事件集成

```java
public class Order {
    private OrderId id;
    private CustomerId customerId;
    private List<OrderItem> items;
    private OrderStatus status;
    private List<DomainEvent> domainEvents = new ArrayList<>();
    
    public void confirm() {
        if (status != OrderStatus.CREATED) {
            throw new IllegalStateException("Order can only be confirmed when in CREATED status");
        }
        
        status = OrderStatus.CONFIRMED;
        domainEvents.add(new OrderConfirmedEvent(id, customerId));
    }
    
    public List<DomainEvent> getDomainEvents() {
        return Collections.unmodifiableList(domainEvents);
    }
    
    public void clearDomainEvents() {
        domainEvents.clear();
    }
}

@Component
public class DomainEventPublisher {
    private final ApplicationEventPublisher eventPublisher;
    
    public DomainEventPublisher(ApplicationEventPublisher eventPublisher) {
        this.eventPublisher = eventPublisher;
    }
    
    public void publishEvents(List<DomainEvent> events) {