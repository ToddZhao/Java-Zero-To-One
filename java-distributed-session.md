# Day 112: Java分布式架构 - 分布式会话管理

## 1. 分布式会话概述

分布式会话是指在分布式系统中，用户的会话信息需要在多个服务器之间共享和同步。这种机制确保了用户在访问分布式系统的不同节点时，能够保持会话的一致性。

## 2. 分布式会话的实现方案

### 2.1 基于Redis的会话管理

```java
@Configuration
@EnableRedisHttpSession
public class RedisSessionConfig {
    
    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory connectionFactory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(connectionFactory);
        template.setKeySerializer(new StringRedisSerializer());
        template.setHashKeySerializer(new StringRedisSerializer());
        template.setHashValueSerializer(new JdkSerializationRedisSerializer());
        template.setValueSerializer(new JdkSerializationRedisSerializer());
        return template;
    }
    
    @Bean
    public RedisConnectionFactory redisConnectionFactory() {
        return new JedisConnectionFactory();
    }
}
```

### 2.2 基于JWT的无状态会话

```java
public class JwtSessionManager {
    private String secretKey;
    private long expiration;

    public String createToken(UserSession session) {
        Date now = new Date();
        Date expiryDate = new Date(now.getTime() + expiration);

        return Jwts.builder()
            .setSubject(session.getUserId())
            .setIssuedAt(now)
            .setExpiration(expiryDate)
            .signWith(SignatureAlgorithm.HS512, secretKey)
            .compact();
    }

    public UserSession validateToken(String token) {
        Claims claims = Jwts.parser()
            .setSigningKey(secretKey)
            .parseClaimsJws(token)
            .getBody();

        UserSession session = new UserSession();
        session.setUserId(claims.getSubject());
        return session;
    }
}
```

## 3. 会话同步机制

### 3.1 会话复制

```java
public class SessionReplicationManager {
    private List<SessionNode> sessionNodes;
    private MessageQueue messageQueue;

    public void replicateSession(String sessionId, Session session) {
        SessionReplicationMessage message = new SessionReplicationMessage(sessionId, session);
        
        // 向其他节点广播会话更新
        for (SessionNode node : sessionNodes) {
            if (!node.isLocal()) {
                messageQueue.send(node.getAddress(), message);
            }
        }
    }

    @MessageListener
    public void handleSessionReplication(SessionReplicationMessage message) {
        // 更新本地会话
        localSessionStore.put(message.getSessionId(), message.getSession());
    }
}
```

### 3.2 会话粘性

```java
public class StickySessionLoadBalancer implements LoadBalancer {
    private ConcurrentMap<String, String> sessionServerMap;

    public Server chooseServer(String sessionId) {
        // 检查是否存在会话亲和性
        String serverId = sessionServerMap.get(sessionId);
        if (serverId != null) {
            Server server = findServer(serverId);
            if (server != null && server.isAlive()) {
                return server;
            }
        }

        // 选择新的服务器
        Server server = selectServer();
        sessionServerMap.put(sessionId, server.getId());
        return server;
    }
}
```

## 4. 会话存储策略

### 4.1 分布式存储

```java
public class DistributedSessionStore implements SessionStore {
    private RedisTemplate redisTemplate;
    private String keyPrefix = "session:";
    private long defaultExpiration = 1800; // 30分钟

    public void save(String sessionId, Session session) {
        String key = keyPrefix + sessionId;
        redisTemplate.opsForValue().set(key, session, defaultExpiration, TimeUnit.SECONDS);
    }

    public Session findById(String sessionId) {
        String key = keyPrefix + sessionId;
        return (Session) redisTemplate.opsForValue().get(key);
    }

    public void delete(String sessionId) {
        String key = keyPrefix + sessionId;
        redisTemplate.delete(key);
    }
}
```

### 4.2 会话数据分片

```java
public class ShardedSessionStore implements SessionStore {
    private List<SessionStore> shards;
    private ConsistentHash<SessionStore> consistentHash;

    public ShardedSessionStore(List<SessionStore> shards) {
        this.shards = shards;
        this.consistentHash = new ConsistentHash<>(shards);
    }

    public void save(String sessionId, Session session) {
        SessionStore shard = consistentHash.locate(sessionId);
        shard.save(sessionId, session);
    }

    public Session findById(String sessionId) {
        SessionStore shard = consistentHash.locate(sessionId);
        return shard.findById(sessionId);
    }
}
```

## 5. 安全性考虑

### 5.1 会话劫持防护

```java
public class SecuritySessionManager {
    private String securityTokenHeader = "X-Security-Token";

    public String createSecurityToken(HttpSession session) {
        String token = generateRandomToken();
        session.setAttribute(securityTokenHeader, token);
        return token;
    }

    public boolean validateSecurityToken(HttpSession session, String token) {
        String storedToken = (String) session.getAttribute(securityTokenHeader);
        return storedToken != null && storedToken.equals(token);
    }

    private String generateRandomToken() {
        return UUID.randomUUID().toString();
    }
}
```

## 6. 性能优化

### 6.1 会话数据压缩

```java
public class CompressedSessionStore implements SessionStore {
    private SessionStore delegate;
    private Compressor compressor;

    public void save(String sessionId, Session session) {
        byte[] compressed = compressor.compress(serialize(session));
        delegate.save(sessionId, compressed);
    }

    public Session findById(String sessionId) {
        byte[] compressed = delegate.findById(sessionId);
        if (compressed != null) {
            byte[] decompressed = compressor.decompress(compressed);
            return deserialize(decompressed);
        }
        return null;
    }
}
```

## 7. 总结

分布式会话管理是构建可扩展分布式系统的关键组件。在实践中需要注意：

1. 根据业务需求选择合适的会话管理方案
2. 确保会话数据的安全性和一致性
3. 实现高效的会话同步机制
4. 优化会话存储和访问性能
5. 考虑系统的可扩展性

## 参考资源

1. Spring Session文档：https://docs.spring.io/spring-session/docs/current/reference/html5/
2. JWT官方文档：https://jwt.io/
3. Redis Session管理：https://redis.io/topics/sessions