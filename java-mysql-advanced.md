# Java 与 MySQL 数据库高级特性教程

## 1. 引言

在掌握了 Java 与 MySQL 的基础操作后，了解一些高级特性可以帮助我们构建更高效、更安全、更健壮的数据库应用。本文将深入探讨 MySQL 的高级特性，并结合 Java 代码展示如何在实际开发中运用这些特性。

## 2. 数据库连接池

数据库连接池是提升应用性能的关键技术。它通过预先创建并维护一组数据库连接，避免了频繁创建和关闭连接的开销。

### 2.1 使用 HikariCP 实现连接池

```java
import com.zaxxer.hikari.HikariConfig;
import com.zaxxer.hikari.HikariDataSource;

public class DatabasePool {
    private static HikariDataSource dataSource;
    
    static {
        // 配置连接池
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:mysql://localhost:3306/your_database");
        config.setUsername("your_username");
        config.setPassword("your_password");
        
        // 连接池优化配置
        config.setMaximumPoolSize(10);           // 最大连接数
        config.setMinimumIdle(5);               // 最小空闲连接
        config.setIdleTimeout(300000);          // 空闲超时时间
        config.setConnectionTimeout(20000);      // 连接超时时间
        
        dataSource = new HikariDataSource(config);
    }
    
    public static Connection getConnection() throws SQLException {
        return dataSource.getConnection();
    }
}
```

## 3. 事务管理

事务管理确保数据库操作的原子性、一致性、隔离性和持久性（ACID）。

### 3.1 实现事务管理器

```java
public class TransactionManager {
    public static void executeInTransaction(TransactionBlock block) throws SQLException {
        Connection conn = null;
        try {
            conn = DatabasePool.getConnection();
            conn.setAutoCommit(false);  // 开启事务
            
            block.execute(conn);        // 执行事务操作
            
            conn.commit();             // 提交事务
        } catch (SQLException e) {
            if (conn != null) {
                conn.rollback();       // 发生异常时回滚
            }
            throw e;
        } finally {
            if (conn != null) {
                conn.setAutoCommit(true);  // 恢复自动提交
                conn.close();
            }
        }
    }
}

// 函数式接口定义事务操作
@FunctionalInterface
interface TransactionBlock {
    void execute(Connection conn) throws SQLException;
}
```

### 3.2 使用事务进行批量操作

```java
public class UserService {
    public void batchCreateUsers(List<User> users) throws SQLException {
        TransactionManager.executeInTransaction(conn -> {
            String sql = "INSERT INTO users (username, email) VALUES (?, ?)";
            try (PreparedStatement pstmt = conn.prepareStatement(sql)) {
                for (User user : users) {
                    pstmt.setString(1, user.getUsername());
                    pstmt.setString(2, user.getEmail());
                    pstmt.addBatch();
                }
                pstmt.executeBatch();
            }
        });
    }
}
```

## 4. 存储过程的使用

存储过程是预编译的 SQL 语句集合，可以提高性能并重用代码。

### 4.1 创建存储过程

```sql
DELIMITER //

CREATE PROCEDURE get_user_stats(IN user_id INT, OUT total_orders INT, OUT total_amount DECIMAL(10,2))
BEGIN
    SELECT COUNT(*), SUM(amount)
    INTO total_orders, total_amount
    FROM orders
    WHERE user_id = user_id;
END //

DELIMITER ;
```

### 4.2 Java 中调用存储过程

```java
public class UserStats {
    public UserStatistics getUserStats(int userId) throws SQLException {
        String sql = "{CALL get_user_stats(?, ?, ?)}";
        
        try (Connection conn = DatabasePool.getConnection();
             CallableStatement stmt = conn.prepareCall(sql)) {
            
            stmt.setInt(1, userId);
            stmt.registerOutParameter(2, Types.INTEGER);
            stmt.registerOutParameter(3, Types.DECIMAL);
            
            stmt.execute();
            
            return new UserStatistics(
                stmt.getInt(2),    // total_orders
                stmt.getBigDecimal(3)  // total_amount
            );
        }
    }
}
```

## 5. 数据库索引优化

### 5.1 创建复合索引

```sql
CREATE INDEX idx_username_email ON users (username, email);
```

### 5.2 使用索引的查询示例

```java
public class UserDAO {
    public List<User> searchUsers(String username, String email) {
        String sql = "SELECT * FROM users WHERE username = ? AND email = ? " +
                    "USE INDEX (idx_username_email)";
        
        List<User> users = new ArrayList<>();
        try (Connection conn = DatabasePool.getConnection();
             PreparedStatement pstmt = conn.prepareStatement(sql)) {
            
            pstmt.setString(1, username);
            pstmt.setString(2, email);
            
            try (ResultSet rs = pstmt.executeQuery()) {
                while (rs.next()) {
                    users.add(mapResultSetToUser(rs));
                }
            }
        } catch (SQLException e) {
            e.printStackTrace();
        }
        return users;
    }
}
```

## 6. 数据库锁机制

### 6.1 实现悲观锁

```java
public class InventoryService {
    public boolean updateStock(int productId, int quantity) throws SQLException {
        TransactionManager.executeInTransaction(conn -> {
            // 使用 SELECT ... FOR UPDATE 加锁
            String selectSql = "SELECT stock FROM inventory WHERE product_id = ? FOR UPDATE";
            String updateSql = "UPDATE inventory SET stock = ? WHERE product_id = ?";
            
            try (PreparedStatement selectStmt = conn.prepareStatement(selectSql)) {
                selectStmt.setInt(1, productId);
                ResultSet rs = selectStmt.executeQuery();
                
                if (rs.next()) {
                    int currentStock = rs.getInt("stock");
                    if (currentStock >= quantity) {
                        // 更新库存
                        try (PreparedStatement updateStmt = conn.prepareStatement(updateSql)) {
                            updateStmt.setInt(1, currentStock - quantity);
                            updateStmt.setInt(2, productId);
                            updateStmt.executeUpdate();
                        }
                    }
                }
            }
        });
        return true;
    }
}
```

### 6.2 实现乐观锁

```java
public class OrderService {
    public boolean updateOrder(Order order) throws SQLException {
        String sql = "UPDATE orders SET status = ?, version = ? " +
                    "WHERE id = ? AND version = ?";
        
        try (Connection conn = DatabasePool.getConnection();
             PreparedStatement pstmt = conn.prepareStatement(sql)) {
            
            pstmt.setString(1, order.getStatus());
            pstmt.setInt(2, order.getVersion() + 1);
            pstmt.setInt(3, order.getId());
            pstmt.setInt(4, order.getVersion());
            
            int updatedRows = pstmt.executeUpdate();
            return updatedRows > 0;
        }
    }
}
```

## 7. 性能监控与优化

### 7.1 SQL 执行时间监控

```java
public class SQLMonitor {
    public static <T> T executeWithTimer(String operation, SQLSupplier<T> supplier) 
            throws SQLException {
        long startTime = System.currentTimeMillis();
        try {
            return supplier.get();
        } finally {
            long endTime = System.currentTimeMillis();
            System.out.printf("Operation '%s' took %d ms%n", 
                            operation, (endTime - startTime));
        }
    }
}

@FunctionalInterface
interface SQLSupplier<T> {
    T get() throws SQLException;
}
```

### 7.2 查询优化示例

```java
public class QueryOptimizer {
    public List<User> getActiveUsers() throws SQLException {
        return SQLMonitor.executeWithTimer("Get Active Users", () -> {
            String sql = """
                SELECT u.*, COUNT(o.id) as order_count
                FROM users u
                LEFT JOIN orders o ON u.id = o.user_id
                WHERE u.status = 'active'
                GROUP BY u.id
                HAVING order_count > 0
                """;
            
            List<User> users = new ArrayList<>();
            try (Connection conn = DatabasePool.getConnection();
                 Statement stmt = conn.createStatement()) {
                
                ResultSet rs = stmt.executeQuery(sql);
                while (rs.next()) {
                    users.add(mapResultSetToUser(rs));
                }
            }
            return users;
        });
    }
}
```

## 8. 最佳实践建议

1. **连接池配置**
   - 根据实际负载调整连接池大小
   - 设置适当的超时时间
   - 定期监控连接池状态

2. **事务管理**
   - 合理设置事务边界
   - 避免长事务
   - 使用适当的隔离级别

3. **索引优化**
   - 为常用查询创建合适的索引
   - 避免过度索引
   - 定期分析索引使用情况

4. **性能优化**
   - 使用批处理提高插入效率
   - 适当使用存储过程
   - 定期维护数据库统计信息

## 9. 总结

本文详细介绍了 MySQL 的多个高级特性，包括连接池管理、事务处理、存储过程使用、索引优化、锁机制实现等。通过合理运用这些特性，我们可以构建出更高效、更可靠的数据库应用。在实际开发中，需要根据具体场景选择合适的特性和优化策略。
