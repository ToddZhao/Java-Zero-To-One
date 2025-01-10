# Java NoSQL 数据库完全指南

## 1. NoSQL 数据库概述

NoSQL（Not Only SQL）数据库是为了解决传统关系型数据库在处理大规模、高并发、非结构化数据时的局限性而产生的。让我们通过一个具体的场景来理解它的优势：

想象你正在开发一个社交媒体平台。用户的个人资料、发帖、评论和社交关系都需要存储。使用传统的关系型数据库，你可能需要创建多个表并进行复杂的关联查询。而使用 NoSQL 数据库，你可以更自然地建模这些数据，并获得更好的性能和扩展性。

## 2. NoSQL 数据库类型

### 2.1 文档型数据库（Document Store）

以 MongoDB 为代表的文档型数据库将数据存储为 JSON 格式的文档。让我们看看如何使用 Java 操作 MongoDB：

```java
import com.mongodb.client.MongoClients;
import com.mongodb.client.MongoClient;
import com.mongodb.client.MongoDatabase;
import org.bson.Document;

public class MongoDBExample {
    private final MongoClient mongoClient;
    private final MongoDatabase database;
    
    public MongoDBExample() {
        // 建立连接
        mongoClient = MongoClients.create("mongodb://localhost:27017");
        database = mongoClient.getDatabase("socialMedia");
    }
    
    // 创建用户个人资料
    public void createUserProfile(UserProfile profile) {
        Document doc = new Document()
            .append("userId", profile.getId())
            .append("username", profile.getUsername())
            .append("email", profile.getEmail())
            .append("interests", profile.getInterests())
            .append("followers", profile.getFollowers())
            .append("following", profile.getFollowing());
            
        database.getCollection("users").insertOne(doc);
    }
    
    // 查找用户及其所有帖子
    public UserWithPosts findUserWithPosts(String userId) {
        Document user = database.getCollection("users")
            .find(new Document("userId", userId))
            .first();
            
        List<Document> posts = database.getCollection("posts")
            .find(new Document("userId", userId))
            .into(new ArrayList<>());
            
        return mapToUserWithPosts(user, posts);
    }
}
```

### 2.2 键值存储（Key-Value Store）

Redis 是最流行的键值存储数据库，特别适合缓存和会话管理：

```java
import redis.clients.jedis.Jedis;
import redis.clients.jedis.JedisPool;

public class RedisExample {
    private final JedisPool jedisPool;
    
    public RedisExample() {
        // 创建连接池
        jedisPool = new JedisPool("localhost", 6379);
    }
    
    // 缓存用户会话
    public void cacheUserSession(String sessionId, UserSession session) {
        try (Jedis jedis = jedisPool.getResource()) {
            // 将会话数据序列化为 JSON
            String sessionJson = objectMapper.writeValueAsString(session);
            
            // 设置会话数据，30分钟过期
            jedis.setex(sessionId, 1800, sessionJson);
        }
    }
    
    // 获取缓存的会话
    public UserSession getUserSession(String sessionId) {
        try (Jedis jedis = jedisPool.getResource()) {
            String sessionJson = jedis.get(sessionId);
            if (sessionJson != null) {
                return objectMapper.readValue(sessionJson, UserSession.class);
            }
            return null;
        }
    }
}
```

### 2.3 列族存储（Column-Family Store）

Apache Cassandra 是一个高度可扩展的列族存储数据库：

```java
import com.datastax.driver.core.*;

public class CassandraExample {
    private final Session session;
    private final Cluster cluster;
    
    public CassandraExample() {
        // 连接到 Cassandra 集群
        cluster = Cluster.builder()
            .addContactPoint("localhost")
            .build();
        session = cluster.connect("socialMedia");
    }
    
    // 存储用户活动日志
    public void logUserActivity(String userId, String activity, long timestamp) {
        PreparedStatement stmt = session.prepare(
            "INSERT INTO user_activities (user_id, timestamp, activity) " +
            "VALUES (?, ?, ?)"
        );
        
        session.execute(stmt.bind(userId, timestamp, activity));
    }
    
    // 获取用户最近活动
    public List<UserActivity> getRecentActivities(String userId, int limit) {
        PreparedStatement stmt = session.prepare(
            "SELECT timestamp, activity FROM user_activities " +
            "WHERE user_id = ? ORDER BY timestamp DESC LIMIT ?"
        );
        
        ResultSet results = session.execute(stmt.bind(userId, limit));
        return mapToUserActivities(results);
    }
}
```

### 2.4 图数据库（Graph Database）

Neo4j 是一个强大的图数据库，特别适合处理复杂的关系数据：

```java
import org.neo4j.driver.*;

public class Neo4jExample {
    private final Driver driver;
    
    public Neo4jExample() {
        driver = GraphDatabase.driver(
            "bolt://localhost:7687",
            AuthTokens.basic("neo4j", "password")
        );
    }
    
    // 创建用户关系
    public void createFriendship(String user1Id, String user2Id) {
        try (Session session = driver.session()) {
            session.writeTransaction(tx -> {
                tx.run(
                    "MATCH (a:User {userId: $user1}), (b:User {userId: $user2}) " +
                    "CREATE (a)-[:FRIENDS_WITH]->(b)",
                    Values.parameters("user1", user1Id, "user2", user2Id)
                );
                return null;
            });
        }
    }
    
    // 查找共同好友
    public List<User> findMutualFriends(String user1Id, String user2Id) {
        try (Session session = driver.session()) {
            return session.readTransaction(tx -> {
                Result result = tx.run(
                    "MATCH (a:User {userId: $user1})-[:FRIENDS_WITH]->(m:User)" +
                    "<-[:FRIENDS_WITH]-(b:User {userId: $user2}) " +
                    "RETURN m",
                    Values.parameters("user1", user1Id, "user2", user2Id)
                );
                
                return result.list(record -> mapToUser(record.get("m").asNode()));
            });
        }
    }
}
```

## 3. NoSQL 数据库的选择策略

选择合适的 NoSQL 数据库需要考虑以下因素：

1. **数据结构**：
   - 文档型：适合半结构化数据，如用户资料、博客文章
   - 键值存储：适合简单的键值对数据，如缓存、会话
   - 列族存储：适合写密集型应用，如日志记录、时间序列数据
   - 图数据库：适合关系密集型数据，如社交网络、推荐系统

2. **性能需求**：
   - 读写比例
   - 延迟要求
   - 并发访问量

3. **扩展性要求**：
   - 数据量增长预期
   - 地理分布需求
   - 一致性要求

## 4. 实战示例：构建社交媒体后端

让我们看一个综合运用多种 NoSQL 数据库的实际例子：

```java
public class SocialMediaService {
    private final MongoDBExample mongodb;    // 存储用户资料和帖子
    private final RedisExample redis;        // 缓存和计数器
    private final Neo4jExample neo4j;        // 社交关系图
    private final CassandraExample cassandra; // 活动日志
    
    public SocialMediaService() {
        this.mongodb = new MongoDBExample();
        this.redis = new RedisExample();
        this.neo4j = new Neo4jExample();
        this.cassandra = new CassandraExample();
    }
    
    // 发布新帖子
    public void createPost(Post post) {
        // 存储帖子内容到 MongoDB
        mongodb.createPost(post);
        
        // 更新用户帖子计数（Redis）
        redis.incrementPostCount(post.getUserId());
        
        // 记录用户活动（Cassandra）
        cassandra.logUserActivity(
            post.getUserId(),
            "Created post: " + post.getId(),
            System.currentTimeMillis()
        );
    }
    
    // 获取推荐好友
    public List<User> getRecommendedFriends(String userId) {
        // 从图数据库获取二度好友
        List<User> recommendedUsers = neo4j.findFriendsOfFriends(userId);
        
        // 从 MongoDB 获取用户详细信息
        List<UserProfile> userProfiles = mongodb.getUserProfiles(
            recommendedUsers.stream()
                .map(User::getId)
                .collect(Collectors.toList())
        );
        
        // 使用 Redis 缓存结果
        redis.cacheRecommendations(userId, userProfiles);
        
        return mapToUsers(userProfiles);
    }
}
```

## 5. 性能优化和最佳实践

### 5.1 连接池管理

```java
public class ConnectionManager {
    private static final JedisPool redisPool;
    private static final MongoClient mongoClient;
    
    static {
        // Redis 连接池配置
        JedisPoolConfig poolConfig = new JedisPoolConfig();
        poolConfig.setMaxTotal(20);
        poolConfig.setMaxIdle(5);
        poolConfig.setMinIdle(1);
        redisPool = new JedisPool(poolConfig, "localhost", 6379);
        
        // MongoDB 连接池配置
        MongoClientSettings settings = MongoClientSettings.builder()
            .applyConnectionString(new ConnectionString("mongodb://localhost:27017"))
            .applyToConnectionPoolSettings(builder -> 
                builder.maxSize(20)
                      .minSize(5)
                      .maxWaitTime(2, TimeUnit.SECONDS))
            .build();
        mongoClient = MongoClients.create(settings);
    }
}
```

### 5.2 数据一致性管理

```java
public class ConsistencyManager {
    private final RedisExample redis;
    private final MongoDBExample mongodb;
    
    // 实现最终一致性
    public void updateUserProfile(String userId, UserProfile profile) {
        // 首先更新主数据库
        mongodb.updateProfile(userId, profile);
        
        // 异步更新缓存
        CompletableFuture.runAsync(() -> {
            try {
                redis.invalidateUserCache(userId);
                redis.cacheUserProfile(userId, profile);
            } catch (Exception e) {
                // 记录错误，稍后重试
                errorQueue.offer(new UpdateTask(userId, profile));
            }
        });
    }
}
```

## 6. 总结

NoSQL 数据库为不同的数据存储需求提供了灵活的解决方案。在实际应用中，我们通常需要:

1. 根据数据特征选择合适的 NoSQL 数据库
2. 合理设计数据模型，避免反模式
3. 正确处理连接池和资源管理
4. 实现适当的一致性策略
5. 持续监控和优化性能

每种 NoSQL 数据库都有其独特的优势，关键是要根据具体的业务需求做出正确的选择和组合。在实践中，可能需要同时使用多种数据库来满足不同的业务场景需求。
