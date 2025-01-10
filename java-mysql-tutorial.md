# Java 与 MySQL 数据库基础教程

## 1. 引言

在现代应用程序开发中，数据库操作是一个不可或缺的部分。Java 作为企业级应用的主流语言，与 MySQL 这样的关系型数据库的结合使用极其普遍。本文将介绍 Java 如何与 MySQL 数据库进行交互，并通过实例来展示基本的数据库操作。

## 2. MySQL 数据库基础

MySQL 是一个开源的关系型数据库管理系统（RDBMS），它使用结构化查询语言（SQL）进行数据库操作。以下是一些关键概念：

- 数据库（Database）：相关表的集合
- 表（Table）：数据的结构化存储单元
- 列（Column）：表中的字段
- 行（Row）：表中的记录
- 主键（Primary Key）：唯一标识表中每条记录的字段

## 3. Java 连接 MySQL 的准备工作

### 3.1 添加 JDBC 依赖

如果你使用 Maven，将以下依赖添加到 pom.xml：

```xml
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>8.0.28</version>
</dependency>
```

### 3.2 数据库连接信息

以下是连接 MySQL 数据库需要的基本信息：

```java
String url = "jdbc:mysql://localhost:3306/your_database";
String username = "your_username";
String password = "your_password";
```

## 4. Java 操作 MySQL 示例

### 4.1 建立数据库连接

```java
import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.SQLException;

public class DatabaseConnection {
    private static final String URL = "jdbc:mysql://localhost:3306/your_database";
    private static final String USERNAME = "your_username";
    private static final String PASSWORD = "your_password";
    
    public static Connection getConnection() throws SQLException {
        return DriverManager.getConnection(URL, USERNAME, PASSWORD);
    }
}
```

### 4.2 创建用户表并插入数据

```java
public class UserDAO {
    // 创建用户表
    public void createUserTable() {
        String sql = """
            CREATE TABLE IF NOT EXISTS users (
                id INT PRIMARY KEY AUTO_INCREMENT,
                username VARCHAR(50) NOT NULL,
                email VARCHAR(100) NOT NULL,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            )
            """;
            
        try (Connection conn = DatabaseConnection.getConnection();
             Statement stmt = conn.createStatement()) {
            stmt.execute(sql);
            System.out.println("Users table created successfully");
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }
    
    // 插入用户数据
    public void insertUser(String username, String email) {
        String sql = "INSERT INTO users (username, email) VALUES (?, ?)";
        
        try (Connection conn = DatabaseConnection.getConnection();
             PreparedStatement pstmt = conn.prepareStatement(sql)) {
            pstmt.setString(1, username);
            pstmt.setString(2, email);
            pstmt.executeUpdate();
            System.out.println("User inserted successfully");
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }
}
```

### 4.3 查询用户数据

```java
public class UserDAO {
    // 查询所有用户
    public List<User> getAllUsers() {
        List<User> users = new ArrayList<>();
        String sql = "SELECT * FROM users";
        
        try (Connection conn = DatabaseConnection.getConnection();
             Statement stmt = conn.createStatement();
             ResultSet rs = stmt.executeQuery(sql)) {
            
            while (rs.next()) {
                User user = new User();
                user.setId(rs.getInt("id"));
                user.setUsername(rs.getString("username"));
                user.setEmail(rs.getString("email"));
                user.setCreatedAt(rs.getTimestamp("created_at"));
                users.add(user);
            }
        } catch (SQLException e) {
            e.printStackTrace();
        }
        
        return users;
    }
    
    // 根据ID查询用户
    public User getUserById(int id) {
        String sql = "SELECT * FROM users WHERE id = ?";
        
        try (Connection conn = DatabaseConnection.getConnection();
             PreparedStatement pstmt = conn.prepareStatement(sql)) {
            
            pstmt.setInt(1, id);
            ResultSet rs = pstmt.executeQuery();
            
            if (rs.next()) {
                User user = new User();
                user.setId(rs.getInt("id"));
                user.setUsername(rs.getString("username"));
                user.setEmail(rs.getString("email"));
                user.setCreatedAt(rs.getTimestamp("created_at"));
                return user;
            }
        } catch (SQLException e) {
            e.printStackTrace();
        }
        
        return null;
    }
}
```

### 4.4 更新和删除用户数据

```java
public class UserDAO {
    // 更新用户信息
    public void updateUser(User user) {
        String sql = "UPDATE users SET username = ?, email = ? WHERE id = ?";
        
        try (Connection conn = DatabaseConnection.getConnection();
             PreparedStatement pstmt = conn.prepareStatement(sql)) {
            
            pstmt.setString(1, user.getUsername());
            pstmt.setString(2, user.getEmail());
            pstmt.setInt(3, user.getId());
            
            int rowsAffected = pstmt.executeUpdate();
            System.out.println(rowsAffected + " row(s) updated");
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }
    
    // 删除用户
    public void deleteUser(int id) {
        String sql = "DELETE FROM users WHERE id = ?";
        
        try (Connection conn = DatabaseConnection.getConnection();
             PreparedStatement pstmt = conn.prepareStatement(sql)) {
            
            pstmt.setInt(1, id);
            int rowsAffected = pstmt.executeUpdate();
            System.out.println(rowsAffected + " row(s) deleted");
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }
}
```

## 5. 使用示例

```java
public class Main {
    public static void main(String[] args) {
        UserDAO userDAO = new UserDAO();
        
        // 创建用户表
        userDAO.createUserTable();
        
        // 插入用户
        userDAO.insertUser("张三", "zhangsan@example.com");
        userDAO.insertUser("李四", "lisi@example.com");
        
        // 查询所有用户
        List<User> users = userDAO.getAllUsers();
        for (User user : users) {
            System.out.println("User: " + user.getUsername() + ", Email: " + user.getEmail());
        }
        
        // 更新用户
        User user = userDAO.getUserById(1);
        if (user != null) {
            user.setEmail("zhangsan_new@example.com");
            userDAO.updateUser(user);
        }
        
        // 删除用户
        userDAO.deleteUser(2);
    }
}
```

## 6. 最佳实践

1. **使用连接池**：在实际应用中，应使用连接池（如 HikariCP、Druid）来管理数据库连接，提高性能。

2. **预处理语句**：始终使用 PreparedStatement 而不是 Statement 来防止 SQL 注入攻击。

3. **资源关闭**：使用 try-with-resources 语句确保数据库资源被正确关闭。

4. **异常处理**：合理处理 SQL 异常，避免异常信息暴露给最终用户。

5. **事务管理**：对于需要保证数据一致性的操作，使用事务管理。

## 7. 总结

本文介绍了 Java 与 MySQL 数据库交互的基础知识，包括：
- 数据库连接的建立
- 基本的 CRUD（创建、读取、更新、删除）操作
- 预处理语句的使用
- 异常处理
- 最佳实践建议

通过这些基础知识，你可以开始构建自己的数据库应用程序。随着应用程序的复杂度增加，可以考虑使用 MyBatis、Hibernate 等 ORM 框架来简化数据库操作。
