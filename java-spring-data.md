# Day 20: Java Spring框架 - 数据访问与ORM

## 引言

Spring框架提供了强大的数据访问支持，包括JDBC抽象、ORM集成和事务管理。本文将详细介绍Spring数据访问层的核心概念和实践应用。

## 1. Spring JDBC

### 1.1 JdbcTemplate

```java
@Repository
public class UserDaoJdbcImpl implements UserDao {
    @Autowired
    private JdbcTemplate jdbcTemplate;
    
    @Override
    public User findById(Long id) {
        String sql = "SELECT * FROM users WHERE id = ?";
        return jdbcTemplate.queryForObject(sql,
            new BeanPropertyRowMapper<>(User.class), id);
    }
    
    @Override
    public List<User> findAll() {
        String sql = "SELECT * FROM users";
        return jdbcTemplate.query(sql,
            new BeanPropertyRowMapper<>(User.class));
    }
    
    @Override
    public void save(User user) {
        String sql = "INSERT INTO users (username, email) VALUES (?, ?)";
        jdbcTemplate.update(sql, user.getUsername(), user.getEmail());
    }
}
```

### 1.2 NamedParameterJdbcTemplate

```java
@Repository
public class ProductDaoJdbcImpl implements ProductDao {
    @Autowired
    private NamedParameterJdbcTemplate namedTemplate;
    
    @Override
    public List<Product> findByCategory(String category, BigDecimal minPrice) {
        String sql = "SELECT * FROM products WHERE category = :category " +
                    "AND price >= :minPrice";
        
        MapSqlParameterSource params = new MapSqlParameterSource()
            .addValue("category", category)
            .addValue("minPrice", minPrice);
        
        return namedTemplate.query(sql, params,
            new BeanPropertyRowMapper<>(Product.class));
    }
}
```

## 2. Spring ORM

### 2.1 JPA配置

```java
@Configuration
@EnableJpaRepositories("com.example.repository")
public class JpaConfig {
    @Bean
    public LocalContainerEntityManagerFactoryBean entityManagerFactory(
            DataSource dataSource) {
        LocalContainerEntityManagerFactoryBean em = 
            new LocalContainerEntityManagerFactoryBean();
        em.setDataSource(dataSource);
        em.setPackagesToScan("com.example.entity");
        
        JpaVendorAdapter vendorAdapter = new HibernateJpaVendorAdapter();
        em.setJpaVendorAdapter(vendorAdapter);
        
        Properties properties = new Properties();
        properties.setProperty("hibernate.hbm2ddl.auto", "update");
        properties.setProperty("hibernate.dialect", 
            "org.hibernate.dialect.MySQL8Dialect");
        em.setJpaProperties(properties);
        
        return em;
    }
    
    @Bean
    public PlatformTransactionManager transactionManager(
            EntityManagerFactory emf) {
        return new JpaTransactionManager(emf);
    }
}
```

### 2.2 实体类

```java
@Entity
@Table(name = "users")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(nullable = false, unique = true)
    private String username;
    
    @Email
    private String email;
    
    @OneToMany(mappedBy = "user", cascade = CascadeType.ALL)
    private List<Order> orders;
    
    // getters and setters
}

@Entity
@Table(name = "orders")
public class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @ManyToOne
    @JoinColumn(name = "user_id")
    private User user;
    
    @Column(nullable = false)
    private BigDecimal amount;
    
    @Temporal(TemporalType.TIMESTAMP)
    private Date orderDate;
    
    // getters and setters
}
```

### 2.3 Repository接口

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByUsername(String username);
    
    @Query("SELECT u FROM User u WHERE u.email LIKE %:domain")
    List<User> findByEmailDomain(@Param("domain") String domain);
    
    @Modifying
    @Query("UPDATE User u SET u.email = :email WHERE u.id = :id")
    int updateEmail(@Param("id") Long id, @Param("email") String email);
}

@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {
    List<Order> findByUserOrderByOrderDateDesc(User user);
    
    @Query("SELECT o FROM Order o WHERE o.amount > :amount")
    List<Order> findLargeOrders(@Param("amount") BigDecimal amount);
}
```

## 3. 事务管理

### 3.1 声明式事务

```java
@Service
@Transactional
public class UserServiceImpl implements UserService {
    @Autowired
    private UserRepository userRepository;
    
    @Autowired
    private OrderRepository orderRepository;
    
    @Override
    @Transactional(readOnly = true)
    public User findById(Long id) {
        return userRepository.findById(id)
            .orElseThrow(() -> new UserNotFoundException(id));
    }
    
    @Override
    @Transactional(rollbackFor = Exception.class)
    public void createUserWithOrders(User user, List<Order> orders) {
        userRepository.save(user);
        orders.forEach(order -> {
            order.setUser(user);
            orderRepository.save(order);
        });
    }
}
```

### 3.2 编程式事务

```java
@Service
public class TransactionService {
    @Autowired
    private PlatformTransactionManager transactionManager;
    
    public void executeInTransaction(TransactionCallback action) {
        TransactionTemplate template = 
            new TransactionTemplate(transactionManager);
        template.execute(status -> {
            try {
                action.execute();
                return null;
            } catch (Exception e) {
                status.setRollbackOnly();
                throw new RuntimeException(e);
            }
        });
    }
}
```

## 4. 实践案例

### 4.1 分页查询

```java
@Service
public class ProductService {
    @Autowired
    private ProductRepository productRepository;
    
    public Page<Product> findProducts(String category, 
            int page, int size, String sortBy) {
        Sort sort = Sort.by(sortBy).descending();
        PageRequest pageRequest = PageRequest.of(page, size, sort);
        
        return productRepository.findByCategory(category, pageRequest);
    }
}

@Repository
public interface ProductRepository 
        extends JpaRepository<Product, Long> {
    Page<Product> findByCategory(String category, Pageable pageable);
    
    @Query(value = "SELECT p FROM Product p WHERE p.price > :price",
           countQuery = "SELECT COUNT(p) FROM Product p WHERE p.price > :price")
    Page<Product> findExpensiveProducts(@Param("price") BigDecimal price,
                                      Pageable pageable);
}
```

### 4.2 批量处理

```java
@Service
public class BatchService {
    @Autowired
    private EntityManager entityManager;
    
    @Transactional
    public void batchInsert(List<Product> products) {
        for (int i = 0; i < products.size(); i++) {
            entityManager.persist(products.get(i));
            if (i % 50 == 0) {
                entityManager.flush();
                entityManager.clear();
            }
        }
    }
    
    @Transactional
    public void batchUpdate(List<Product> products) {
        for (int i = 0; i < products.size(); i++) {
            entityManager.merge(products.get(i));
            if (i % 50 == 0) {
                entityManager.flush();
                entityManager.clear();
            }
        }
    }
}
```

## 5. 最佳实践

1. 使用合适的数据访问技术
2. 正确配置事务管理
3. 优化数据库查询
4. 使用批量操作提高性能
5. 实现适当的异常处理

## 总结

本文介绍了Spring数据访问层的核心概念和实践应用，包括：

1. Spring JDBC的使用
2. JPA和Hibernate集成
3. 事务管理机制
4. 数据访问最佳实践
5. 实践案例

通过掌握这些知识，我们可以更好地使用Spring框架进行数据访问和ORM操作。

## 参考资源

1. Spring Data JPA文档
2. Hibernate用户指南
3. Spring事务管理指南
4. JPA最佳实践