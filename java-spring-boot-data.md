# Day 23: Java Spring Boot数据访问

## 引言

Spring Boot提供了强大的数据访问支持，包括Spring Data JPA、Spring Data JDBC和Spring Data MongoDB等模块。本文将详细介绍Spring Boot数据访问的核心概念和实践应用。

## 1. Spring Data JPA

### 1.1 基本配置

```yaml
# application.yml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/mydb
    username: root
    password: password
  jpa:
    hibernate:
      ddl-auto: update
    show-sql: true
    properties:
      hibernate:
        format_sql: true
        dialect: org.hibernate.dialect.MySQL8Dialect
```

### 1.2 实体类和仓库

```java
@Entity
@Table(name = "products")
public class Product {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(nullable = false)
    private String name;
    
    @Column(precision = 10, scale = 2)
    private BigDecimal price;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "category_id")
    private Category category;
    
    // getters and setters
}

@Repository
public interface ProductRepository extends JpaRepository<Product, Long> {
    List<Product> findByCategory(Category category);
    
    @Query("SELECT p FROM Product p WHERE p.price > :minPrice")
    List<Product> findExpensiveProducts(@Param("minPrice") BigDecimal minPrice);
    
    @Modifying
    @Query("UPDATE Product p SET p.price = :newPrice WHERE p.id = :id")
    int updatePrice(@Param("id") Long id, @Param("newPrice") BigDecimal newPrice);
}
```

## 2. Spring Data JDBC

### 2.1 配置和使用

```java
@Configuration
public class JdbcConfig {
    @Bean
    public NamedParameterJdbcTemplate jdbcTemplate(DataSource dataSource) {
        return new NamedParameterJdbcTemplate(dataSource);
    }
}

@Repository
public class ProductJdbcRepository {
    private final NamedParameterJdbcTemplate jdbcTemplate;
    
    public ProductJdbcRepository(NamedParameterJdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }
    
    public List<Product> findAll() {
        String sql = "SELECT * FROM products";
        return jdbcTemplate.query(sql, new BeanPropertyRowMapper<>(Product.class));
    }
    
    public Optional<Product> findById(Long id) {
        String sql = "SELECT * FROM products WHERE id = :id";
        MapSqlParameterSource params = new MapSqlParameterSource("id", id);
        try {
            return Optional.ofNullable(jdbcTemplate.queryForObject(sql, 
                params, new BeanPropertyRowMapper<>(Product.class)));
        } catch (EmptyResultDataAccessException e) {
            return Optional.empty();
        }
    }
}
```

## 3. 事务管理

### 3.1 声明式事务

```java
@Service
@Transactional
public class ProductService {
    private final ProductRepository productRepository;
    private final CategoryRepository categoryRepository;
    
    public ProductService(ProductRepository productRepository,
                         CategoryRepository categoryRepository) {
        this.productRepository = productRepository;
        this.categoryRepository = categoryRepository;
    }
    
    @Transactional(readOnly = true)
    public List<Product> findAllProducts() {
        return productRepository.findAll();
    }
    
    @Transactional(rollbackFor = Exception.class)
    public Product createProduct(Product product) {
        if (product.getCategory() != null && 
            product.getCategory().getId() != null) {
            Category category = categoryRepository
                .findById(product.getCategory().getId())
                .orElseThrow(() -> new EntityNotFoundException(
                    "Category not found"));
            product.setCategory(category);
        }
        return productRepository.save(product);
    }
}
```

## 4. 数据审计

### 4.1 审计配置

```java
@Configuration
@EnableJpaAuditing
public class AuditConfig {
    @Bean
    public AuditorAware<String> auditorProvider() {
        return () -> Optional.ofNullable(SecurityContextHolder.getContext())
            .map(SecurityContext::getAuthentication)
            .map(Authentication::getName);
    }
}

@EntityListeners(AuditingEntityListener.class)
@MappedSuperclass
public abstract class Auditable {
    @CreatedDate
    @Column(updatable = false)
    private LocalDateTime createdAt;
    
    @LastModifiedDate
    private LocalDateTime updatedAt;
    
    @CreatedBy
    @Column(updatable = false)
    private String createdBy;
    
    @LastModifiedBy
    private String updatedBy;
    
    // getters and setters
}

@Entity
public class Product extends Auditable {
    // 实体属性
}
```

## 5. 数据分页和排序

### 5.1 分页查询

```java
@RestController
@RequestMapping("/api/products")
public class ProductController {
    private final ProductService productService;
    
    public ProductController(ProductService productService) {
        this.productService = productService;
    }
    
    @GetMapping
    public Page<Product> getProducts(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "10") int size,
            @RequestParam(defaultValue = "id") String sortBy) {
        return productService.findProducts(PageRequest.of(
            page, size, Sort.by(sortBy)));
    }
}

@Service
public class ProductService {
    private final ProductRepository productRepository;
    
    public Page<Product> findProducts(Pageable pageable) {
        return productRepository.findAll(pageable);
    }
    
    public Page<Product> findProductsByCategory(
            Category category, Pageable pageable) {
        return productRepository.findByCategory(category, pageable);
    }
}
```

## 6. 缓存支持

### 6.1 缓存配置

```java
@Configuration
@EnableCaching
public class CacheConfig {
    @Bean
    public CacheManager cacheManager() {
        SimpleCacheManager cacheManager = new SimpleCacheManager();
        cacheManager.setCaches(Arrays.asList(
            new ConcurrentMapCache("products"),
            new ConcurrentMapCache("categories")
        ));
        return cacheManager;
    }
}

@Service
public class ProductService {
    @Cacheable(value = "products", key = "#id")
    public Product findById(Long id) {
        return productRepository.findById(id)
            .orElseThrow(() -> new EntityNotFoundException(
                "Product not found"));
    }
    
    @CacheEvict(value = "products", key = "#id")
    public void deleteProduct(Long id) {
        productRepository.deleteById(id);
    }
    
    @CachePut(value = "products", key = "#result.id")
    public Product updateProduct(Product product) {
        return productRepository.save(product);
    }
}
```

## 7. 最佳实践

1. 使用合适的数据访问技术
2. 正确配置事务管理
3. 实现数据审计
4. 优化查询性能
5. 合理使用缓存

## 总结

本文介绍了Spring Boot数据访问的核心概念和实践应用，包括：

1. Spring Data JPA的使用
2. Spring Data JDBC的应用
3. 事务管理机制
4. 数据审计功能
5. 分页和缓存支持

通过掌握这些知识，我们可以更好地使用Spring Boot进行数据访问操作，提高应用程序的性能和可维护性。

## 参考资源

1. Spring Boot官方文档
2. Spring Data JPA参考指南
3. Spring Data JDBC文档
4. Spring Boot缓存指南