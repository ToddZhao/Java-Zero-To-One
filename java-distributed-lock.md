# Day 110: Java分布式架构 - 分布式锁设计

## 1. 分布式锁概述

分布式锁是控制分布式系统之间同步访问共享资源的一种方式。在分布式系统中，为了保证数据的一致性，我们需要保证在同一时间只有一个客户端可以对共享资源进行操作。

## 2. 分布式锁的实现方案

### 2.1 基于Redis的分布式锁

```java
public class RedisDistributedLock {
    private StringRedisTemplate redisTemplate;
    private String lockKey;
    private String lockValue;
    private long expireTime;

    public RedisDistributedLock(StringRedisTemplate redisTemplate, String lockKey) {
        this.redisTemplate = redisTemplate;
        this.lockKey = lockKey;
        this.lockValue = UUID.randomUUID().toString();
        this.expireTime = 30000; // 30秒过期
    }

    public boolean tryLock() {
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

### 2.2 基于Zookeeper的分布式锁

```java
public class ZookeeperDistributedLock {
    private final ZooKeeper zk;
    private final String lockPath;
    private String currentNode;

    public ZookeeperDistributedLock(String connectString, String lockPath) throws IOException {
        this.zk = new ZooKeeper(connectString, 3000, null);
        this.lockPath = lockPath;
    }

    public boolean tryLock() throws Exception {
        // 创建临时顺序节点
        currentNode = zk.create(lockPath + "/lock_", new byte[0],
            ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.EPHEMERAL_SEQUENTIAL);

        // 获取所有子节点
        List<String> children = zk.getChildren(lockPath, false);
        Collections.sort(children);

        // 判断当前节点是否为最小节点
        String smallestNode = children.get(0);
        return currentNode.endsWith(smallestNode);
    }

    public void unlock() throws Exception {
        zk.delete(currentNode, -1);
    }
}
```

### 2.3 基于数据库的分布式锁

```java
@Service
public class DatabaseDistributedLock {
    @Autowired
    private JdbcTemplate jdbcTemplate;

    public boolean tryLock(String resourceId, String owner, long expireTime) {
        try {
            String sql = "INSERT INTO distributed_lock (resource_id, owner, expire_time) " +
                         "VALUES (?, ?, ?)";
            jdbcTemplate.update(sql, resourceId, owner, 
                new Timestamp(System.currentTimeMillis() + expireTime));
            return true;
        } catch (DuplicateKeyException e) {
            // 清理过期锁
            cleanExpiredLock(resourceId);
            return false;
        }
    }

    private void cleanExpiredLock(String resourceId) {
        String sql = "DELETE FROM distributed_lock WHERE resource_id = ? " +
                     "AND expire_time < ?";
        jdbcTemplate.update(sql, resourceId, new Timestamp(System.currentTimeMillis()));
    }

    public void unlock(String resourceId, String owner) {
        String sql = "DELETE FROM distributed_lock WHERE resource_id = ? AND owner = ?";
        jdbcTemplate.update(sql, resourceId, owner);
    }
}
```

## 3. Redisson分布式锁框架

```java
public class RedissonLockExample {
    private RedissonClient redisson;

    public void processWithLock() {
        RLock lock = redisson.getLock("myLock");
        try {
            // 尝试加锁，最多等待100秒，上锁后10秒自动解锁
            boolean isLocked = lock.tryLock(100, 10, TimeUnit.SECONDS);
            if (isLocked) {
                try {
                    // 处理业务逻辑
                    businessOperation();
                } finally {
                    lock.unlock();
                }
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }

    private void businessOperation() {
        // 业务处理逻辑
    }
}
```

## 4. 最佳实践

### 4.1 锁的粒度控制

```java
public class LockGranularityExample {
    private DistributedLock lock;

    public void processOrder(String orderId) {
        // 使用订单ID作为锁的粒度，而不是锁定整个订单表
        String lockKey = "order:" + orderId;
        try {
            if (lock.tryLock(lockKey)) {
                try {
                    // 处理订单逻辑
                    processOrderDetails(orderId);
                } finally {
                    lock.unlock(lockKey);
                }
            }
        } catch (Exception e) {
            // 异常处理
        }
    }
}
```

### 4.2 锁超时和续期

```java
public class LockWatchDog {
    private ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(1);
    private DistributedLock lock;

    public void lockWithRenewal(String lockKey) {
        try {
            if (lock.tryLock(lockKey)) {
                // 启动续期任务
                scheduler.scheduleAtFixedRate(() -> {
                    try {
                        lock.renewal(lockKey);
                    } catch (Exception e) {
                        // 续期失败处理
                    }
                }, 10, 10, TimeUnit.SECONDS);

                try {
                    // 执行业务逻辑
                    businessOperation();
                } finally {
                    scheduler.shutdown();
                    lock.unlock(lockKey);
                }
            }
        } catch (Exception e) {
            // 异常处理
        }
    }
}
```

## 5. 性能优化

### 5.1 读写锁分离

```java
public class ReadWriteLockExample {
    private RedissonClient redisson;

    public void processWithReadWriteLock() {
        RReadWriteLock rwlock = redisson.getReadWriteLock("myLock");
        
        // 读操作
        RLock readLock = rwlock.readLock();
        try {
            readLock.lock();
            // 读取操作
        } finally {
            readLock.unlock();
        }

        // 写操作
        RLock writeLock = rwlock.writeLock();
        try {
            writeLock.lock();
            // 写入操作
        } finally {
            writeLock.unlock();
        }
    }
}
```

## 6. 总结

分布式锁是分布式系统中的一个重要组件，主要用于解决分布式环境下的资源竞争问题。在实际应用中：

1. 根据业务场景选择合适的实现方案（Redis、Zookeeper、数据库）
2. 注意锁的粒度控制，避免锁粒度过大影响性能
3. 考虑锁的超时和续期机制，防止死锁
4. 在读多写少的场景下，考虑使用读写锁分离提升性能

## 参考资源

1. Redisson官方文档：https://redisson.org/
2. ZooKeeper官方文档：https://zookeeper.apache.org/
3. Redis分布式锁：https://redis.io/topics/distlock