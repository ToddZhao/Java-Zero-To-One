# Day 108: Java分布式系统进阶 - 分布式系统设计与实践

分布式系统是现代大规模应用不可或缺的架构模式。本文将深入探讨分布式系统的核心概念、设计原则、关键技术以及最佳实践，帮助开发者构建可靠、可扩展的分布式系统。

## 1. 分布式系统基础

### 1.1 CAP理论

CAP理论是分布式系统设计的基础理论，它指出在分布式系统中，一致性（Consistency）、可用性（Availability）和分区容错性（Partition tolerance）这三个特性最多只能同时满足其中两个。

```java
public class CAPDemo {
    public interface DistributedSystem {
        // 一致性：所有节点在同一时间具有相同的数据
        boolean ensureConsistency();
        
        // 可用性：服务一直可用，但不保证获取的数据为最新
        boolean ensureAvailability();
        
        // 分区容错性：分布式系统在遇到网络分区故障时仍能继续运行
        boolean ensurePartitionTolerance();
    }
    
    // CP系统示例
    public class CPSystem implements DistributedSystem {
        @Override
        public boolean ensureConsistency() {
            // 实现强一致性
            return true;
        }
        
        @Override
        public boolean ensureAvailability() {
            // 在网络分区时可能牺牲可用性
            return false;
        }
        
        @Override
        public boolean ensurePartitionTolerance() {
            // 保证分区容错
            return true;
        }
    }
    
    // AP系统示例
    public class APSystem implements DistributedSystem {
        @Override
        public boolean ensureConsistency() {
            // 采用最终一致性
            return false;
        }
        
        @Override
        public boolean ensureAvailability() {
            // 保证系统可用
            return true;
        }
        
        @Override
        public boolean ensurePartitionTolerance() {
            // 保证分区容错
            return true;
        }
    }
}
```

### 1.2 BASE理论

BASE理论是对CAP中一致性和可用性权衡的结果：

- **基本可用**（Basically Available）
- **软状态**（Soft State）
- **最终一致性**（Eventually Consistent）

```java
public class BASEDemo {
    public class EventualConsistencySystem {
        private Map<String, String> dataStore;
        private Queue<UpdateEvent> updateQueue;
        
        public void update(String key, String value) {
            // 异步更新，保证基本可用性
            updateQueue.offer(new UpdateEvent(key, value));
            
            // 后台进程处理更新
            processUpdates();
        }
        
        private void processUpdates() {
            new Thread(() -> {
                while (true) {
                    UpdateEvent event = updateQueue.poll();
                    if (event != null) {
                        // 最终一致性：异步更新所有节点
                        updateAllNodes(event);
                    }
                }
            }).start();
        }
    }
}
```

## 2. 分布式系统设计模式

### 2.1 服务发现模式

```java
public class ServiceDiscoveryPattern {
    public interface ServiceRegistry {
        void register(ServiceInstance instance);
        void deregister(ServiceInstance instance);
        List<ServiceInstance> getInstances(String serviceId);
    }
    
    @Data
    @Builder
    public class ServiceInstance {
        private String serviceId;
        private String host;
        private int port;
        private Map<String, String> metadata;
        private ServiceHealth health;
    }
    
    // 使用ZooKeeper实现服务注册
    public class ZookeeperServiceRegistry implements ServiceRegistry {
        private CuratorFramework client;
        private static final String BASE_PATH = "/services";
        
        @Override
        public void register(ServiceInstance instance) {
            String path = BASE_PATH + "/" + instance.getServiceId();
            try {
                // 创建临时节点
                client.create()
                      .creatingParentsIfNeeded()
                      .withMode(CreateMode.EPHEMERAL)
                      .forPath(path, serialize(instance));
            } catch (Exception e) {
                throw new RuntimeException("Failed to register service", e);
            }
        }
    }
}
```

### 2.2 负载均衡模式

```java
public class LoadBalancerPattern {
    public interface LoadBalancer {
        ServiceInstance choose(List<ServiceInstance> instances);
    }
    
    // 轮询负载均衡器
    public class RoundRobinLoadBalancer implements LoadBalancer {
        private AtomicInteger position = new AtomicInteger(0);
        
        @Override
        public ServiceInstance choose(List<ServiceInstance> instances) {
            if (instances.isEmpty()) {
                return null;
            }
            
            int pos = position.incrementAndGet() % instances.size();
            return instances.get(pos);
        }
    }
    
    // 加权负载均衡器
    public class WeightedLoadBalancer implements LoadBalancer {
        @Override
        public ServiceInstance choose(List<ServiceInstance> instances) {
            int totalWeight = instances.stream()
                .mapToInt(inst -> inst.getMetadata().get("weight"))
                .sum();
            
            int randomWeight = ThreadLocalRandom.current().nextInt(totalWeight);
            int currentWeight = 0;
            
            for (ServiceInstance instance : instances) {
                currentWeight += Integer.parseInt(instance.getMetadata().get("weight"));
                if (randomWeight < currentWeight) {
                    return instance;
                }
            }
            
            return instances.get(0);
        }
    }
}
```

### 2.3 熔断器模式

```java
public class CircuitBreakerPattern {
    public class CircuitBreaker {
        private final int failureThreshold;
        private final long resetTimeout;
        private AtomicInteger failureCount;
        private AtomicReference<CircuitState> state;
        private AtomicLong lastFailureTime;
        
        public enum CircuitState {
            CLOSED, OPEN, HALF_OPEN
        }
        
        public CircuitBreaker(int failureThreshold, long resetTimeout) {
            this.failureThreshold = failureThreshold;
            this.resetTimeout = resetTimeout;
            this.failureCount = new AtomicInteger(0);
            this.state = new AtomicReference<>(CircuitState.CLOSED);
            this.lastFailureTime = new AtomicLong(0);
        }
        
        public <T> T execute(Supplier<T> supplier) throws Exception {
            if (isOpen()) {
                throw new CircuitBreakerOpenException();
            }
            
            try {
                T result = supplier.get();
                reset();
                return result;
            } catch (Exception e) {
                recordFailure();
                throw e;
            }
        }
        
        private void recordFailure() {
            failureCount.incrementAndGet();
            lastFailureTime.set(System.currentTimeMillis());
            
            if (failureCount.get() >= failureThreshold) {
                state.set(CircuitState.OPEN);
            }
        }
        
        private boolean isOpen() {
            if (state.get() == CircuitState.OPEN) {
                // 检查是否达到重置时间
                if (System.currentTimeMillis() - lastFailureTime.get() > resetTimeout) {
                    state.set(CircuitState.HALF_OPEN);
                    return false;
                }
                return true;
            }
            return false;
        }
        
        private void reset() {
            failureCount.set(0);
            state.set(CircuitState.CLOSED);
        }
    }
}
```

## 3. 分布式事务

### 3.1 两阶段提交（2PC）

```java
public class TwoPhaseCommitDemo {
    public interface TransactionParticipant {
        boolean prepare();
        void commit();
        void rollback();
    }
    
    public class TransactionCoordinator {
        private List<TransactionParticipant> participants;
        
        public boolean executeTransaction() {
            // 阶段1：准备
            boolean canCommit = true;
            for (TransactionParticipant participant : participants) {
                if (!participant.prepare()) {
                    canCommit = false;
                    break;
                }
            }
            
            // 阶段2：提交或回滚
            if (canCommit) {
                for (TransactionParticipant participant : participants) {
                    try {
                        participant.commit();
                    } catch (Exception e) {
                        // 提交失败，需要人工介入
                        return false;
                    }
                }
                return true;
            } else {
                for (TransactionParticipant participant : participants) {
                    participant.rollback();
                }
                return false;
            }
        }
    }
}
```

### 3.2 TCC（Try-Confirm-Cancel）

```java
public class TCCDemo {
    public interface TCCService {
        boolean try();
        boolean confirm();
        boolean cancel();
    }
    
    public class OrderTCCService implements TCCService {
        private final OrderService orderService;
        private final String orderId;
        
        @Override
        public boolean try() {
            // 尝试阶段：冻结库存
            return orderService.tryFreezeStock(orderId);
        }
        
        @Override
        public boolean confirm() {
            // 确认阶段：扣减库存
            return orderService.confirmDeductStock(orderId);
        }
        
        @Override
        public boolean cancel() {
            // 取消阶段：解冻库存
            return orderService.cancelFreezeStock(orderId);
        }
    }
}
```

## 4. 分布式锁

### 4.1 基于Redis的分布式锁

```java
public class RedisDistributedLock {
    private StringRedisTemplate redisTemplate;
    private String lockKey;
    private String lockValue;
    private long expireTime;
    
    public boolean tryLock() {
        lockValue = UUID.randomUUID().toString();
        return redisTemplate.opsForValue()
            .setIfAbsent(lockKey, lockValue, expireTime, TimeUnit.MILLISECONDS);
    }
    
    public boolean unlock() {
        String script = "if redis.call('get', KEYS[1]) == ARGV[1] then " +
                       "return redis.call('del', KEYS[1]) else return 0 end";
        return redisTemplate.execute(new DefaultRedisScript<>(script, Boolean.class),
            Collections.singletonList(lockKey), lockValue);
    }
}
```

### 4.2 基于ZooKeeper的分布式锁

```java
public class ZookeeperDistributedLock {
    private final CuratorFramework client;
    private final String lockPath;
    private InterProcessMutex lock;
    
    public ZookeeperDistributedLock(CuratorFramework client, String lockPath) {
        this.client = client;
        this.lockPath = lockPath;
        this.lock = new InterProcessMutex(client, lockPath);
    }
    
    public boolean tryLock(long timeout, TimeUnit unit) throws Exception {
        return lock.acquire(timeout, unit);
    }
    
    public void unlock() throws Exception {
        lock.release();
    }
}
```

## 5. 分布式缓存

### 5.1 多级缓存架构

```java
public class MultiLevelCache {
    public interface Cache<K, V> {
        V get(K key);
        void put(K key, V value);
        void remove(K key);
    }
    
    public class LocalCache<K, V> implements Cache<K, V> {
        private LoadingCache<K, V> cache;
        
        public LocalCache(long maximumSize, long expireAfterWrite) {
            cache = Caffeine.newBuilder()
                .maximumSize(maximumSize)
                .expireAfterWrite(expireAfterWrite, TimeUnit.SECONDS)
                .build(key -> null);
        }
        
        @Override
        public V get(K key) {
            return cache.getIfPresent(key);
        }
        
        @Override
        public void put(K key, V value) {
            cache.put(key, value);
        }
        
        @Override
        public void remove(K key) {
            cache.invalidate(key);
        }
    }
    
    public class RedisCache<K, V> implements Cache<K, V> {
        private StringRedisTemplate redisTemplate;
        private String prefix;
        private Class<V> type;
        
        @Override
        public V get(K key) {
            String json = redisTemplate.opsForValue().get(prefix + key);
            return json != null ? JSON.parseObject(json, type) : null;
        }
        
        @Override
        public void put(K key, V value) {
            redisTemplate.opsForValue().set(prefix + key,