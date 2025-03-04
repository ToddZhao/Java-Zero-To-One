# Day 11: Java数据库连接与JDBC

## 引言

JDBC（Java Database Connectivity）是Java语言中用于执行SQL语句的标准API，它为Java应用程序提供了统一的数据库访问方式。本文将详细介绍JDBC的核心概念和实践应用。

## 1. JDBC基础

### 1.1 JDBC架构

JDBC API包含两个包：
- java.sql：核心JDBC API
- javax.sql：JDBC扩展API

### 1.2 建立数据库连接

```java
public class DatabaseConnection {
    private static final String URL = "jdbc:mysql://localhost:3306/mydb";
    private static final String USERNAME = "root";
    private static final String PASSWORD = "password";
    
    public static Connection getConnection() throws SQLException {
        return DriverManager.getConnection(URL, USERNAME, PASSWORD);
    }
    
    public static void closeConnection(Connection conn) {
        if (conn != null) {
            try {
                conn.close();
            } catch (SQLException e) {
                e.printStackTrace();
            }
        }
    }
}
```

## 2. JDBC操作

### 2.1 执行SQL语句

```java
public class JdbcTemplate {
    // 执行查询
    public static <T> List<T> query(String sql, RowMapper<T> rowMapper, 
            Object... params) {
        List<T> results = new ArrayList<>();
        
        try (Connection conn = DatabaseConnection.getConnection();
             PreparedStatement stmt = conn.prepareStatement(sql)) {
            
            // 设置参数
            for (int i = 0; i < params.length; i++) {
                stmt.setObject(i + 1, params[i]);
            }
            
            // 执行查询
            ResultSet rs = stmt.executeQuery();
            while (rs.next()) {
                results.add(rowMapper.mapRow(rs));
            }
            
        } catch (SQLException e) {
            e.printStackTrace();
        }
        
        return results;
    }
    
    // 执行更新
    public static int update(String sql, Object... params) {
        try (Connection conn = DatabaseConnection.getConnection();
             PreparedStatement stmt = conn.prepareStatement(sql)) {
            
            // 设置参数
            for (int i = 0; i < params.length; i++) {
                stmt.setObject(i + 1, params[i]);
            }
            
            return stmt.executeUpdate();
            
        } catch (SQLException e) {
            e.printStackTrace();
            return 0;
        }
    }
}

// 行映射接口
@FunctionalInterface
public interface RowMapper<T> {
    T mapRow(ResultSet rs) throws SQLException;
}
```

### 2.2 事务管理

```java
public class TransactionManager {
    public static void executeInTransaction(Connection conn, 
            TransactionCallback callback) throws SQLException {
        boolean autoCommit = conn.getAutoCommit();
        try {
            conn.setAutoCommit(false);
            callback.execute(conn);
            conn.commit();
        } catch (SQLException e) {
            conn.rollback();
            throw e;
        } finally {
            conn.setAutoCommit(autoCommit);
        }
    }
}

@FunctionalInterface
public interface TransactionCallback {
    void execute(Connection conn) throws SQLException;
}
```

## 3. 数据访问对象模式

### 3.1 DAO接口和实现

```java
// 实体类
public class User {
    private Long id;
    private String username;
    private String email;
    
    // getters and setters
}

// DAO接口
public interface UserDao {
    User findById(Long id);
    List<User> findAll();
    void save(User user);
    void update(User user);
    void delete(Long id);
}

// DAO实现
public class UserDaoImpl implements UserDao {
    private static final String SQL_FIND_BY_ID = 
        "SELECT * FROM users WHERE id = ?";
    private static final String SQL_FIND_ALL = 
        "SELECT * FROM users";
    private static final String SQL_INSERT = 
        "INSERT INTO users (username, email) VALUES (?, ?)";
    private static final String SQL_UPDATE = 
        "UPDATE users SET username = ?, email = ? WHERE id = ?";
    private static final String SQL_DELETE = 
        "DELETE FROM users WHERE id = ?";
    
    @Override
    public User findById(Long id) {
        return JdbcTemplate.query(SQL_FIND_BY_ID, rs -> {
            User user = new User();
            user.setId(rs.getLong("id"));
            user.setUsername(rs.getString("username"));
            user.setEmail(rs.getString("email"));
            return user;
        }, id).stream().findFirst().orElse(null);
    }
    
    @Override
    public List<User> findAll() {
        return JdbcTemplate.query(SQL_FIND_ALL, rs -> {
            User user = new User();
            user.setId(rs.getLong("id"));
            user.setUsername(rs.getString("username"));
            user.setEmail(rs.getString("email"));
            return user;
        });
    }
    
    @Override
    public void save(User user) {
        JdbcTemplate.update(SQL_INSERT, 
            user.getUsername(), user.getEmail());
    }
    
    @Override
    public void update(User user) {
        JdbcTemplate.update(SQL_UPDATE, 
            user.getUsername(), user.getEmail(), user.getId());
    }
    
    @Override
    public void delete(Long id) {
        JdbcTemplate.update(SQL_DELETE, id);
    }
}
```

## 4. 连接池

### 4.1 使用连接池

```java
public class ConnectionPool {
    private static ComboPooledDataSource dataSource;
    
    static {
        try {
            dataSource = new ComboPooledDataSource();
            dataSource.setDriverClass("com.mysql.cj.jdbc.Driver");
            dataSource.setJdbcUrl("jdbc:mysql://localhost:3306/mydb");
            dataSource.setUser("root");
            dataSource.setPassword("password");
            
            // 连接池配置
            dataSource.setInitialPoolSize(5);
            dataSource.setMinPoolSize(5);
            dataSource.setMaxPoolSize(20);
            dataSource.setMaxIdleTime(300);
            dataSource.setMaxStatements(50);
            
        } catch (Exception e) {
            throw new RuntimeException("初始化连接池失败", e);
        }
    }
    
    public static Connection getConnection() throws SQLException {
        return dataSource.getConnection();
    }
}
```

## 5. 实践案例

### 5.1 批量处理

```java
public class BatchProcessor {
    public static void batchInsert(List<User> users) {
        String sql = "INSERT INTO users (username, email) VALUES (?, ?)";
        
        try (Connection conn = ConnectionPool.getConnection();
             PreparedStatement stmt = conn.prepareStatement(sql)) {
            
            conn.setAutoCommit(false);
            
            for (User user : users) {
                stmt.setString(1, user.getUsername());
                stmt.setString(2, user.getEmail());
                stmt.addBatch();
            }
            
            stmt.executeBatch();
            conn.commit();
            
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }
}
```

### 5.2 存储过程调用

```java
public class StoredProcedureExample {
    public static List<User> getUsersByDepartment(String department) {
        List<User> users = new ArrayList<>();
        String sql = "{CALL get_users_by_department(?)}"; 
        
        try (Connection conn = ConnectionPool.getConnection();
             CallableStatement stmt = conn.prepareCall(sql)) {
            
            stmt.setString(1, department);
            ResultSet rs = stmt.executeQuery();
            
            while (rs.next()) {
                User user = new User();
                user.setId(rs.getLong("id"));
                user.setUsername(rs.getString("username"));
                user.setEmail(rs.getString("email"));
                users.add(user);
            }
            
        } catch (SQLException e) {
            e.printStackTrace();
        }
        
        return users;
    }
}
```

## 6. 最佳实践

1. 使用连接池管理数据库连接
2. 始终使用PreparedStatement防止SQL注入
3. 正确管理事务
4. 及时释放数据库资源
5. 使用批处理提高性能

## 总结

本文介绍了Java数据库连接与JDBC的核心概念和实践应用，包括：

1. JDBC基础知识
2. 数据库操作方法
3. DAO设计模式
4. 连接池使用
5. 实践案例和最佳实践

通过掌握这些知识，我们可以更好地进行数据库编程，提高应用程序的性能和可维护性。

## 参考资源

1. Java官方文档：https://docs.oracle.com/javase/tutorial/jdbc/
2. MySQL官方文档
3. JDBC最佳实践指南
4. 数据库连接池使用指南