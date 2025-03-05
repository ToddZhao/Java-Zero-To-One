# Day 109: Java分布式架构 - 分布式事务实现

## 1. 什么是分布式事务

分布式事务是指事务的参与者、支持事务的服务器、资源服务器以及事务管理器分别位于不同的分布式系统的不同节点之上。简单的说，就是一次大的操作由不同的小操作组成，这些小的操作分布在不同的服务器上，且属于不同的应用，分布式事务需要保证这些小操作要么全部成功，要么全部失败。

## 2. 分布式事务的特性（ACID）

- 原子性（Atomicity）：事务是一个原子操作单元，要么全部执行，要么全部不执行
- 一致性（Consistency）：事务执行前后，数据保持一致
- 隔离性（Isolation）：并发执行的事务之间不会互相影响
- 持久性（Durability）：事务一旦提交，其结果就是永久性的

## 3. 分布式事务的实现方案

### 3.1 两阶段提交（2PC）

```java
public class TwoPhaseCommit {
    private TransactionManager transactionManager;
    private List<ResourceManager> resourceManagers;
    
    public void executeTransaction() {
        // 阶段1：准备阶段
        boolean canCommit = true;
        for (ResourceManager rm : resourceManagers) {
            if (!rm.prepare()) {
                canCommit = false;
                break;
            }
        }
        
        // 阶段2：提交/回滚阶段
        if (canCommit) {
            for (ResourceManager rm : resourceManagers) {
                rm.commit();
            }
        } else {
            for (ResourceManager rm : resourceManagers) {
                rm.rollback();
            }
        }
    }
}
```

### 3.2 TCC（Try-Confirm-Cancel）

```java
@Service
public class TCCService {
    
    @Transactional
    public void transfer(Account from, Account to, BigDecimal amount) {
        // Try阶段
        try {
            // 尝试冻结转出账户金额
            boolean tryResult = from.tryFreeze(amount);
            if (!tryResult) {
                throw new RuntimeException("资金冻结失败");
            }
            
            // 尝试预增加转入账户金额
            tryResult = to.tryIncrease(amount);
            if (!tryResult) {
                throw new RuntimeException("预增加金额失败");
            }
            
            // Confirm阶段
            from.confirm();
            to.confirm();
            
        } catch (Exception e) {
            // Cancel阶段
            from.cancel();
            to.cancel();
            throw e;
        }
    }
}
```

### 3.3 SAGA模式

```java
public class SagaTransaction {
    private List<SagaStep> steps;
    private List<SagaStep> compensationSteps;
    
    public void execute() {
        try {
            // 执行正向操作
            for (SagaStep step : steps) {
                step.execute();
            }
        } catch (Exception e) {
            // 补偿操作
            for (SagaStep step : compensationSteps) {
                step.compensate();
            }
        }
    }
}
```

## 4. 分布式事务框架

### 4.1 Seata

```java
@GlobalTransactional
@Service
public class OrderService {
    
    @Autowired
    private AccountService accountService;
    
    @Autowired
    private InventoryService inventoryService;
    
    public void createOrder(String userId, String commodityCode, int count) {
        // 创建订单
        Order order = new Order();
        order.setUserId(userId);
        order.setCommodityCode(commodityCode);
        order.setCount(count);
        orderMapper.insert(order);
        
        // 扣减库存
        inventoryService.deduct(commodityCode, count);
        
        // 扣减账户余额
        accountService.debit(userId, order.getMoney());
    }
}
```

## 5. 最佳实践

### 5.1 事务补偿

```java
public class CompensationManager {
    private Map<String, CompensationTask> compensationTasks;
    
    public void registerCompensation(String transactionId, CompensationTask task) {
        compensationTasks.put(transactionId, task);
    }
    
    public void compensate(String transactionId) {
        CompensationTask task = compensationTasks.get(transactionId);
        if (task != null) {
            task.execute();
        }
    }
}
```

### 5.2 幂等性设计

```java
public class IdempotentService {
    
    @Autowired
    private RedisTemplate redisTemplate;
    
    public void processWithIdempotency(String key, Runnable business) {
        String lockKey = "idempotent:" + key;
        try {
            // 通过Redis实现幂等性控制
            Boolean success = redisTemplate.opsForValue().setIfAbsent(lockKey, "1", 24, TimeUnit.HOURS);
            if (success != null && success) {
                business.run();
            }
        } finally {
            redisTemplate.delete(lockKey);
        }
    }
}
```

## 6. 性能优化

### 6.1 异步补偿

```java
public class AsyncCompensationService {
    
    @Autowired
    private ThreadPoolExecutor compensationExecutor;
    
    public void asyncCompensate(String transactionId) {
        compensationExecutor.execute(() -> {
            try {
                // 执行补偿逻辑
                compensate(transactionId);
            } catch (Exception e) {
                // 补偿失败处理
                handleCompensationFailure(transactionId, e);
            }
        });
    }
}
```

## 7. 总结

分布式事务是分布式系统中的一个重要且复杂的问题。在实际应用中，需要根据业务场景选择合适的分布式事务解决方案：

1. 对于强一致性要求的场景，可以选择2PC
2. 对于性能要求高的场景，可以选择TCC或SAGA
3. 在实现时要注意考虑幂等性、补偿机制和性能优化

## 参考资源

1. Seata官方文档：http://seata.io/
2. 分布式事务理论：https://www.jdon.com/distributed.html
3. Spring分布式事务：https://docs.spring.io/spring-framework/docs/current/reference/html/data-access.html#transaction