# Day 121: Java架构设计 - DDD领域驱动设计

## 1. DDD概述

DDD（Domain-Driven Design，领域驱动设计）是一种软件开发方法论，它将复杂的业务需求转化为软件模型，通过领域模型来指导软件设计和开发。DDD强调业务领域专家和开发团队的紧密协作，共同建立统一的领域模型。

## 2. DDD核心概念

### 2.1 领域模型

```java
// 值对象
public class Money {
    private final BigDecimal amount;
    private final String currency;
    
    public Money(BigDecimal amount, String currency) {
        this.amount = amount;
        this.currency = currency;
    }
    
    public Money add(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new IllegalArgumentException("Cannot add different currencies");
        }
        return new Money(this.amount.add(other.amount), this.currency);
    }
    
    // 值对象的相等性比较
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        Money money = (Money) o;
        return amount.equals(money.amount) && currency.equals(money.currency);
    }
}

// 实体
@Entity
public class Order {
    @Id
    private OrderId id;
    private CustomerId customerId;
    private List<OrderLine> orderLines;
    private OrderStatus status;
    private Money totalAmount;
    
    public void addOrderLine(Product product, int quantity) {
        OrderLine line = new OrderLine(product, quantity);
        orderLines.add(line);
        recalculateTotal();
    }
    
    public void confirm() {
        if (status != OrderStatus.DRAFT) {
            throw new IllegalStateException("Order can only be confirmed when in DRAFT status");
        }
        status = OrderStatus.CONFIRMED;
        // 发布领域事件
        DomainEvents.publish(new OrderConfirmedEvent(this.id));
    }
}
```

### 2.2 限界上下文

```java
// 订单上下文
public interface OrderRepository {
    Order findById(OrderId id);
    void save(Order order);
}

// 库存上下文
public interface InventoryRepository {
    Inventory findByProductId(ProductId id);
    void save(Inventory inventory);
}

// 上下文映射
public class OrderToInventoryService {
    @Autowired
    private InventoryRepository inventoryRepository;
    
    @EventListener
    public void handleOrderConfirmed(OrderConfirmedEvent event) {
        Order order = orderRepository.findById(event.getOrderId());
        for (OrderLine line : order.getOrderLines()) {
            Inventory inventory = inventoryRepository.findByProductId(line.getProductId());
            inventory.reserve(line.getQuantity());
            inventoryRepository.save(inventory);
        }
    }
}
```

## 3. DDD战术设计模式

### 3.1 聚合根

```java
@Aggregate
public class Customer {
    @AggregateId
    private CustomerId id;
    private String name;
    private List<Address> addresses;
    private List<Order> orders;
    
    public void placeOrder(Order order) {
        validateOrderLimit();
        orders.add(order);
        DomainEvents.publish(new OrderPlacedEvent(order.getId()));
    }
    
    private void validateOrderLimit() {
        if (orders.size() >= 100) {
            throw new DomainException("Customer has reached maximum order limit");
        }
    }
}
```

### 3.2 领域服务

```java
public interface OrderDomainService {
    Order createOrder(CustomerId customerId, List<OrderLineDto> orderLines);
    void processPayment(OrderId orderId, Money amount);
}

@Service
public class OrderDomainServiceImpl implements OrderDomainService {
    @Autowired
    private OrderRepository orderRepository;
    @Autowired
    private CustomerRepository customerRepository;
    @Autowired
    private PaymentService paymentService;
    
    @Override
    @Transactional
    public Order createOrder(CustomerId customerId, List<OrderLineDto> orderLines) {
        Customer customer = customerRepository.findById(customerId);
        Order order = new Order(customer);
        
        for (OrderLineDto lineDto : orderLines) {
            order.addOrderLine(lineDto.getProduct(), lineDto.getQuantity());
        }
        
        orderRepository.save(order);
        return order;
    }
    
    @Override
    @Transactional
    public void processPayment(OrderId orderId, Money amount) {
        Order order = orderRepository.findById(orderId);
        if (order.getTotalAmount().compareTo(amount) != 0) {
            throw new InvalidPaymentAmountException();
        }
        
        PaymentResult result = paymentService.process(amount);
        if (result.isSuccessful()) {
            order.markAsPaid();
            orderRepository.save(order);
        }
    }
}
```

## 4. DDD实践建议

### 4.1 领域事件

```java
public interface DomainEvent {
    String getEventId();
    Date getOccurredOn();
}

@Value
public class OrderConfirmedEvent implements DomainEvent {
    private final String eventId = UUID.randomUUID().toString();
    private final Date occurredOn = new Date();
    private final OrderId orderId;
    
    public OrderConfirmedEvent(OrderId orderId) {
        this.orderId = orderId;
    }
}

@Component
public class OrderEventHandler {
    @Autowired
    private NotificationService notificationService;
    
    @EventListener
    public void handleOrderConfirmed(OrderConfirmedEvent event) {
        // 发送确认通知
        notificationService.sendOrderConfirmation(event.getOrderId());
    }
}
```

### 4.2 仓储模式

```java
@Repository
public class JpaOrderRepository implements OrderRepository {
    @PersistenceContext
    private EntityManager entityManager;
    
    @Override
    public Order findById(OrderId id) {
        Order order = entityManager.find(Order.class, id);
        if (order == null) {
            throw new OrderNotFoundException(id);
        }
        return order;
    }
    
    @Override
    public void save(Order order) {
        entityManager.persist(order);
    }
    
    @Override
    public List<Order> findByCustomerId(CustomerId customerId) {
        return entityManager.createQuery(
            "SELECT o FROM Order o WHERE o.customerId = :customerId",
            Order.class)
            .setParameter("customerId", customerId)
            .getResultList();
    }
}
```

## 5. 最佳实践

### 5.1 分层架构

```java
// 应用层服务
@Service
public class OrderApplicationService {
    @Autowired
    private OrderDomainService orderDomainService;
    @Autowired
    private OrderRepository orderRepository;
    
    @Transactional
    public OrderId createOrder(CreateOrderCommand command) {
        // 应用层负责协调领域对象
        Order order = orderDomainService.createOrder(
            command.getCustomerId(),
            command.getOrderLines()
        );
        
        // 持久化
        orderRepository.save(order);
        
        return order.getId();
    }
}

// 表现层
@RestController
@RequestMapping("/api/orders")
public class OrderController {
    @Autowired
    private OrderApplicationService orderApplicationService;
    
    @PostMapping
    public ResponseEntity<OrderId> createOrder(@RequestBody CreateOrderRequest request) {
        CreateOrderCommand command = new CreateOrderCommand(
            request.getCustomerId(),
            request.getOrderLines()
        );
        
        OrderId orderId = orderApplicationService.createOrder(command);
        return ResponseEntity.ok(orderId);
    }
}
```

## 参考资源

1. 《领域驱动设计：软件核心复杂性应对之道》- Eric Evans
2. 《实现领域驱动设计》- Vaughn Vernon
3. DDD社区：https://ddd-practitioners.com/
4. Spring DDD示例：https://github.com/spring-projects/spring-data-examples