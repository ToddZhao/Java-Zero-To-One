# Day 122: Java架构设计 - CQRS架构模式

## 1. CQRS概述

CQRS（Command Query Responsibility Segregation，命令查询职责分离）是一种架构模式，它将系统的读操作和写操作分离成两个独立的模型。这种分离可以带来更好的性能、可扩展性和维护性。

## 2. CQRS核心概念

### 2.1 命令模型

```java
// 命令
public class CreateOrderCommand {
    private final CustomerId customerId;
    private final List<OrderLineItem> items;
    private final Money totalAmount;
    
    public CreateOrderCommand(CustomerId customerId, List<OrderLineItem> items, Money totalAmount) {
        this.customerId = customerId;
        this.items = items;
        this.totalAmount = totalAmount;
    }
    
    // getters
}

// 命令处理器
@Service
public class CreateOrderCommandHandler {
    private final OrderRepository orderRepository;
    private final EventBus eventBus;
    
    public CreateOrderCommandHandler(OrderRepository orderRepository, EventBus eventBus) {
        this.orderRepository = orderRepository;
        this.eventBus = eventBus;
    }
    
    @Transactional
    public OrderId handle(CreateOrderCommand command) {
        // 创建订单聚合根
        Order order = new Order(
            command.getCustomerId(),
            command.getItems(),
            command.getTotalAmount()
        );
        
        // 保存订单
        orderRepository.save(order);
        
        // 发布事件
        eventBus.publish(new OrderCreatedEvent(order.getId()));
        
        return order.getId();
    }
}
```

### 2.2 查询模型

```java
// 查询DTO
public class OrderSummaryQuery {
    private final OrderId orderId;
    
    public OrderSummaryQuery(OrderId orderId) {
        this.orderId = orderId;
    }
}

// 查询结果DTO
public class OrderSummaryDTO {
    private final OrderId orderId;
    private final String customerName;
    private final BigDecimal totalAmount;
    private final String status;
    private final List<OrderItemDTO> items;
    
    // constructor and getters
}

// 查询处理器
@Service
public class OrderQueryHandler {
    private final OrderReadRepository orderReadRepository;
    
    public OrderQueryHandler(OrderReadRepository orderReadRepository) {
        this.orderReadRepository = orderReadRepository;
    }
    
    public OrderSummaryDTO handle(OrderSummaryQuery query) {
        return orderReadRepository.getOrderSummary(query.getOrderId());
    }
}
```

## 3. CQRS实现

### 3.1 事件溯源

```java
// 事件
public interface DomainEvent {
    String getEventId();
    Date getOccurredOn();
}

@Value
public class OrderCreatedEvent implements DomainEvent {
    private final String eventId = UUID.randomUUID().toString();
    private final Date occurredOn = new Date();
    private final OrderId orderId;
    private final CustomerId customerId;
    private final List<OrderLineItem> items;
    private final Money totalAmount;
}

// 事件存储
public interface EventStore {
    void saveEvents(String aggregateId, List<DomainEvent> events, int expectedVersion);
    List<DomainEvent> getEvents(String aggregateId);
}

@Repository
public class MongoEventStore implements EventStore {
    private final MongoTemplate mongoTemplate;
    
    @Override
    public void saveEvents(String aggregateId, List<DomainEvent> events, int expectedVersion) {
        events.forEach(event -> {
            EventDocument eventDocument = new EventDocument(aggregateId, event);
            mongoTemplate.save(eventDocument, "events");
        });
    }
    
    @Override
    public List<DomainEvent> getEvents(String aggregateId) {
        Query query = Query.query(Criteria.where("aggregateId").is(aggregateId));
        return mongoTemplate.find(query, EventDocument.class, "events")
            .stream()
            .map(EventDocument::getEvent)
            .collect(Collectors.toList());
    }
}
```

### 3.2 读模型更新

```java
@Service
public class OrderProjection {
    private final OrderReadRepository orderReadRepository;
    
    @EventListener
    public void on(OrderCreatedEvent event) {
        OrderReadModel orderReadModel = new OrderReadModel();
        orderReadModel.setOrderId(event.getOrderId());
        orderReadModel.setCustomerId(event.getCustomerId());
        orderReadModel.setItems(event.getItems());
        orderReadModel.setTotalAmount(event.getTotalAmount());
        orderReadModel.setStatus(OrderStatus.CREATED);
        
        orderReadRepository.save(orderReadModel);
    }
    
    @EventListener
    public void on(OrderPaidEvent event) {
        OrderReadModel orderReadModel = orderReadRepository.findById(event.getOrderId())
            .orElseThrow(() -> new OrderNotFoundException(event.getOrderId()));
        
        orderReadModel.setStatus(OrderStatus.PAID);
        orderReadModel.setPaymentDate(event.getPaymentDate());
        
        orderReadRepository.save(orderReadModel);
    }
}
```

## 4. CQRS最佳实践

### 4.1 命令验证

```java
public interface CommandValidator<T> {
    ValidationResult validate(T command);
}

public class CreateOrderCommandValidator implements CommandValidator<CreateOrderCommand> {
    @Override
    public ValidationResult validate(CreateOrderCommand command) {
        ValidationResult result = new ValidationResult();
        
        if (command.getCustomerId() == null) {
            result.addError("customerId", "Customer ID is required");
        }
        
        if (command.getItems() == null || command.getItems().isEmpty()) {
            result.addError("items", "Order must contain at least one item");
        }
        
        if (command.getTotalAmount() == null || command.getTotalAmount().isNegative()) {
            result.addError("totalAmount", "Total amount must be positive");
        }
        
        return result;
    }
}
```

### 4.2 性能优化

```java
@Configuration
public class CQRSConfig {
    
    @Bean
    public CacheManager cacheManager() {
        SimpleCacheManager cacheManager = new SimpleCacheManager();
        cacheManager.setCaches(Arrays.asList(
            new ConcurrentMapCache("orderSummaries"),
            new ConcurrentMapCache("customerOrders")
        ));
        return cacheManager;
    }
    
    @Bean
    public OrderReadRepository orderReadRepository() {
        return new CachedOrderReadRepository(cacheManager());
    }
}

@Repository
public class CachedOrderReadRepository implements OrderReadRepository {
    private final CacheManager cacheManager;
    
    @Cacheable(value = "orderSummaries", key = "#orderId")
    public OrderSummaryDTO getOrderSummary(OrderId orderId) {
        // 从数据库获取订单信息
        return orderRepository.getOrderSummary(orderId);
    }
    
    @Cacheable(value = "customerOrders", key = "#customerId")
    public List<OrderSummaryDTO> getCustomerOrders(CustomerId customerId) {
        // 从数据库获取客户订单列表
        return orderRepository.getCustomerOrders(customerId);
    }
    
    @CacheEvict(value = {"orderSummaries", "customerOrders"}, allEntries = true)
    public void clearCache() {
        // 清除所有缓存
    }
}
```

## 5. 实践建议

### 5.1 API设计

```java
@RestController
@RequestMapping("/api/orders")
public class OrderController {
    private final CommandBus commandBus;
    private final QueryBus queryBus;
    
    @PostMapping
    public ResponseEntity<OrderId> createOrder(@RequestBody CreateOrderRequest request) {
        CreateOrderCommand command = new CreateOrderCommand(
            request.getCustomerId(),
            request.getItems(),
            request.getTotalAmount()
        );
        
        OrderId orderId = commandBus.send(command);
        return ResponseEntity.ok(orderId);
    }
    
    @GetMapping("/{orderId}")
    public ResponseEntity<OrderSummaryDTO> getOrder(@PathVariable OrderId orderId) {
        OrderSummaryQuery query = new OrderSummaryQuery(orderId);
        OrderSummaryDTO summary = queryBus.send(query);
        return ResponseEntity.ok(summary);
    }
}
```

## 参考资源

1. 《CQRS Journey》- Microsoft patterns & practices
2. 《Implementing Domain-Driven Design》- Vaughn Vernon
3. CQRS模式：https://martinfowler.com/bliki/CQRS.html
4. Axon Framework：https://axoniq.io/