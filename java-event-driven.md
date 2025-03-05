# Day 120: Java微服务高级 - 事件驱动架构

## 1. 事件驱动架构概述

事件驱动架构（Event-Driven Architecture，EDA）是一种软件架构模式，它基于事件的生产、检测、消费和响应。在这种架构中，系统组件之间通过事件进行通信，而不是通过直接调用。事件驱动架构特别适合构建松耦合、可扩展的分布式系统。

## 2. 事件驱动架构的核心概念

### 2.1 事件、生产者和消费者

```java
public class OrderEvent {
    private String eventId;
    private String eventType; // 如 ORDER_CREATED, ORDER_SHIPPED
    private Date timestamp;
    private Order payload;
    
    // 构造函数、getter和setter
}

@Service
public class OrderEventProducer {
    
    @Autowired
    private KafkaTemplate<String, OrderEvent> kafkaTemplate;
    
    public void publishOrderCreatedEvent(Order order) {
        OrderEvent event = new OrderEvent();
        event.setEventId(UUID.randomUUID().toString());
        event.setEventType("ORDER_CREATED");
        event.setTimestamp(new Date());
        event.setPayload(order);
        
        kafkaTemplate.send("order-events", order.getId(), event);
    }
}

@Service
public class OrderEventConsumer {
    
    @KafkaListener(topics = "order-events")
    public void handleOrderEvent(OrderEvent event) {
        switch (event.getEventType()) {
            case "ORDER_CREATED":
                processNewOrder(event.getPayload());
                break;
            case "ORDER_SHIPPED":
                updateShippingStatus(event.getPayload());
                break;
            // 处理其他事件类型
        }
    }
    
    private void processNewOrder(Order order) {
        // 处理新订单逻辑
    }
    
    private void updateShippingStatus(Order order) {
        // 更新配送状态逻辑
    }
}
```

### 2.2 事件存储和事件溯源

```java
public interface EventStore {
    void saveEvent(DomainEvent event);
    List<DomainEvent> getEventsForAggregate(String aggregateId);
}

@Repository
public class MongoEventStore implements EventStore {
    
    @Autowired
    private MongoTemplate mongoTemplate;
    
    @Override
    public void saveEvent(DomainEvent event) {
        mongoTemplate.save(event, "events");
    }
    
    @Override
    public List<DomainEvent> getEventsForAggregate(String aggregateId) {
        Query query = new Query(Criteria.where("aggregateId").is(aggregateId));
        query.with(Sort.by(Sort.Direction.ASC, "sequenceNumber"));
        return mongoTemplate.find(query, DomainEvent.class, "events");
    }
}

public abstract class Aggregate {
    protected String id;
    protected long version;
    
    public void loadFromHistory(List<DomainEvent> events) {
        events.forEach(this::apply);
    }
    
    protected abstract void apply(DomainEvent event);
}

public class OrderAggregate extends Aggregate {
    private String customerId;
    private List<OrderItem> items;
    private OrderStatus status;
    
    public OrderAggregate(String id) {
        this.id = id;
        this.items = new ArrayList<>();
    }
    
    public void createOrder(String customerId, List<OrderItem> items) {
        OrderCreatedEvent event = new OrderCreatedEvent(id, version++, customerId, items);
        apply(event);
        DomainEventPublisher.publish(event);
    }
    
    public void shipOrder() {
        if (status != OrderStatus.PAID) {
            throw new IllegalStateException("Order must be paid before shipping");
        }
        
        OrderShippedEvent event = new OrderShippedEvent(id, version++, new Date());
        apply(event);
        DomainEventPublisher.publish(event);
    }
    
    @Override
    protected void apply(DomainEvent event) {
        if (event instanceof OrderCreatedEvent) {
            OrderCreatedEvent e = (OrderCreatedEvent) event;
            this.customerId = e.getCustomerId();
            this.items = e.getItems();
            this.status = OrderStatus.CREATED;
        } else if (event instanceof OrderShippedEvent) {
            this.status = OrderStatus.SHIPPED;
        }
        // 处理其他事件类型
    }
}
```

## 3. 消息中间件集成

### 3.1 Kafka集成

```java
@Configuration
public class KafkaConfig {
    
    @Bean
    public ProducerFactory<String, Object> producerFactory() {
        Map<String, Object> configProps = new HashMap<>();
        configProps.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        configProps.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        configProps.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);
        return new DefaultKafkaProducerFactory<>(configProps);
    }
    
    @Bean
    public KafkaTemplate<String, Object> kafkaTemplate() {
        return new KafkaTemplate<>(producerFactory());
    }
    
    @Bean
    public ConsumerFactory<String, Object> consumerFactory() {
        Map<String, Object> configProps = new HashMap<>();
        configProps.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        configProps.put(ConsumerConfig.GROUP_ID_CONFIG, "order-service");
        configProps.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        configProps.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, JsonDeserializer.class);
        configProps.put(JsonDeserializer.TRUSTED_PACKAGES, "com.example.domain.event");
        return new DefaultKafkaConsumerFactory<>(configProps);
    }
    
    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, Object> kafkaListenerContainerFactory() {
        ConcurrentKafkaListenerContainerFactory<String, Object> factory = 
            new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory());
        return factory;
    }
}
```

### 3.2 RabbitMQ集成

```java
@Configuration
public class RabbitMQConfig {
    
    @Bean
    public TopicExchange orderExchange() {
        return new TopicExchange("order-exchange");
    }
    
    @Bean
    public Queue orderCreatedQueue() {
        return new Queue("order-created-queue", true);
    }
    
    @Bean
    public Queue orderShippedQueue() {
        return new Queue("order-shipped-queue", true);
    }
    
    @Bean
    public Binding orderCreatedBinding(Queue orderCreatedQueue, TopicExchange orderExchange) {
        return BindingBuilder.bind(orderCreatedQueue)
            .to(orderExchange)
            .with("order.created");
    }
    
    @Bean
    public Binding orderShippedBinding(Queue orderShippedQueue, TopicExchange orderExchange) {
        return BindingBuilder.bind(orderShippedQueue)
            .to(orderExchange)
            .with("order.shipped");
    }
    
    @Bean
    public MessageConverter jsonMessageConverter() {
        return new Jackson2JsonMessageConverter();
    }
    
    @Bean
    public RabbitTemplate rabbitTemplate(ConnectionFactory connectionFactory) {
        RabbitTemplate template = new RabbitTemplate(connectionFactory);
        template.setMessageConverter(jsonMessageConverter());
        return template;
    }
}

@Service
public class OrderEventPublisher {
    
    @Autowired
    private RabbitTemplate rabbitTemplate;
    
    public void publishOrderCreatedEvent(Order order) {
        OrderCreatedEvent event = new OrderCreatedEvent(order.getId(), 0, order.getCustomerId(), order.getItems());
        rabbitTemplate.convertAndSend("order-exchange", "order.created", event);
    }
    
    public void publishOrderShippedEvent(Order order) {
        OrderShippedEvent event = new OrderShippedEvent(order.getId(), order.getVersion(), new Date());
        rabbitTemplate.convertAndSend("order-exchange", "order.shipped", event);
    }
}

@Service
public class OrderEventListener {
    
    @RabbitListener(queues = "order-created-queue")
    public void handleOrderCreatedEvent(OrderCreatedEvent event) {
        // 处理订单创建事件
    }
    
    @RabbitListener(queues = "order-shipped-queue")
    public void handleOrderShippedEvent(OrderShippedEvent event) {
        // 处理订单发货事件
    }
}
```

## 4. 事件驱动微服务实现

### 4.1 CQRS模式

```java
// 命令部分
public class CreateOrderCommand {
    private String customerId;
    private List<OrderItem> items;
    
    // 构造函数、getter和setter
}

@Service
public class OrderCommandService {
    
    @Autowired
    private EventStore eventStore;
    
    @Autowired
    private OrderEventPublisher eventPublisher;
    
    @Transactional
    public String createOrder(CreateOrderCommand command) {
        String orderId = UUID.randomUUID().toString();
        OrderAggregate order = new OrderAggregate(orderId);
        order.createOrder(command.getCustomerId(), command.getItems());
        
        // 保存事件并发布
        OrderCreatedEvent event = new OrderCreatedEvent(orderId, 0, command.getCustomerId(), command.getItems());
        eventStore.saveEvent(event);
        eventPublisher.publishOrderCreatedEvent(event);
        
        return orderId;
    }
}

// 查询部分
@Service
public class OrderQueryService {
    
    @Autowired
    private OrderRepository orderRepository;
    
    public OrderDTO getOrder(String orderId) {
        Order order = orderRepository.findById(orderId)
            .orElseThrow(() -> new OrderNotFoundException(orderId));
        return mapToDTO(order);
    }
    
    public List<OrderDTO> getOrdersByCustomer(String customerId) {
        return orderRepository.findByCustomerId(customerId).stream()
            .map(this::mapToDTO)
            .collect(Collectors.toList());
    }
    
    private OrderDTO mapToDTO(Order order) {
        // 映射逻辑
        return new OrderDTO(/* ... */);
    }
}

// 事件处理器更新查询模型
@Service
public class OrderEventHandler {
    
    @Autowired
    private OrderRepository orderRepository;
    
    @EventListener
    public void on(OrderCreatedEvent event) {
        Order order = new Order();
        order.setId(event.getAggregateId());
        order.setCustomerId(event.getCustomerId());
        order.setItems(event.getItems());
        order.setStatus(OrderStatus.CREATED);
        order.setCreatedAt(event.getTimestamp());
        
        orderRepository.save(order);
    }
    
    @EventListener
    public void on(OrderShippedEvent event) {
        orderRepository.findById(event.getAggregateId())
            .ifPresent(order -> {
                order.setStatus(OrderStatus.SHIPPED);
                order.setShippedAt(event.getShippingDate());
                orderRepository.save(order);
            });
    }
}
```

### 4.2 Saga模式

```java
public interface SagaStep<T> {
    void execute(T context);
    void compensate(T context);
}

public class OrderSaga {
    
    private final List<SagaStep<OrderContext>> steps;
    
    public OrderSaga() {
        this.steps = new ArrayList<>();
        steps.add(new ValidateInventoryStep());
        steps.add(new ProcessPaymentStep());
        steps.add(new UpdateInventoryStep());
        steps.add(new CreateShipmentStep());
    }
    
    public void execute(OrderContext context) {
        int currentStep = 0;
        try {
            for (; currentStep < steps.size(); currentStep++) {
                steps.get(currentStep).execute(context);
            }
        } catch (Exception e) {
            // 补偿事务
            for (int i = currentStep; i >= 0; i--) {
                try {
                    steps.get(i).compensate(context);
                } catch (Exception ex) {
                    // 记录补偿失败
                }
            }
            throw new SagaExecutionException("Order saga failed", e);
        }
    }
}

public class ValidateInventoryStep implements SagaStep<OrderContext> {
    
    @Autowired
    private InventoryService inventoryService;
    
    @Override
    public void execute(OrderContext context) {
        boolean available = inventoryService.checkAvailability(context.getOrder().getItems());
        if (!available) {
            throw new InsufficientInventoryException();
        }
    }
    
    @Override
    public void compensate(OrderContext context) {
        // 无需补偿，因为只是检查库存
    }
}

p