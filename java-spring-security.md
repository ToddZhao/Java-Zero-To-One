# Day 45: Java安全框架 - Spring Security实战

## 引言

在企业级应用开发中，安全性是一个不可忽视的关键要素。Spring Security作为Java生态系统中最流行的安全框架，提供了全面的认证、授权和防护机制。本文将介绍Spring Security的核心概念和实践应用。

## 1. Spring Security基础

### 1.1 核心概念

- **认证(Authentication)**: 验证用户身份的过程
- **授权(Authorization)**: 确定用户是否有权限执行某操作
- **安全上下文(SecurityContext)**: 存储当前认证用户的信息
- **安全过滤器链(SecurityFilterChain)**: 处理HTTP请求的安全过滤器集合

### 1.2 工作原理

Spring Security基于过滤器链实现安全控制：

1. 请求到达应用
2. 经过一系列过滤器处理(认证、授权等)
3. 通过所有过滤器后，请求才能到达控制器

## 2. Spring Boot集成Spring Security

### 2.1 依赖配置

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

### 2.2 基本配置

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    
    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(authorize -> authorize
                .requestMatchers("/public/**").permitAll()
                .requestMatchers("/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .formLogin(form -> form
                .loginPage("/login")
                .permitAll()
            )
            .logout(logout -> logout
                .logoutSuccessUrl("/login?logout")
                .permitAll()
            );
            
        return http.build();
    }
    
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

## 3. 用户认证实现

### 3.1 基于内存的用户认证

```java
@Configuration
public class InMemoryUserDetailsConfig {
    
    @Bean
    public UserDetailsService userDetailsService(PasswordEncoder encoder) {
        UserDetails user = User.builder()
            .username("user")
            .password(encoder.encode("password"))
            .roles("USER")
            .build();
            
        UserDetails admin = User.builder()
            .username("admin")
            .password(encoder.encode("admin"))
            .roles("USER", "ADMIN")
            .build();
            
        return new InMemoryUserDetailsManager(user, admin);
    }
}
```

### 3.2 基于数据库的用户认证

```java
@Entity
@Table(name = "users")
public class User implements UserDetails {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(unique = true)
    private String username;
    
    private String password;
    
    @ManyToMany(fetch = FetchType.EAGER)
    @JoinTable(
        name = "user_roles",
        joinColumns = @JoinColumn(name = "user_id"),
        inverseJoinColumns = @JoinColumn(name = "role_id")
    )
    private Set<Role> roles = new HashSet<>();
    
    // UserDetails方法实现
    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        return roles.stream()
            .map(role -> new SimpleGrantedAuthority("ROLE_" + role.getName()))
            .collect(Collectors.toList());
    }
    
    @Override
    public boolean isAccountNonExpired() {
        return true;
    }
    
    @Override
    public boolean isAccountNonLocked() {
        return true;
    }
    
    @Override
    public boolean isCredentialsNonExpired() {
        return true;
    }
    
    @Override
    public boolean isEnabled() {
        return true;
    }
    
    // Getters and setters
}
```

```java
@Service
public class UserDetailsServiceImpl implements UserDetailsService {
    
    @Autowired
    private UserRepository userRepository;
    
    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        return userRepository.findByUsername(username)
            .orElseThrow(() -> new UsernameNotFoundException("用户不存在: " + username));
    }
}
```

## 4. 授权控制

### 4.1 基于URL的授权

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    
    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(authorize -> authorize
                .requestMatchers("/", "/home", "/public/**").permitAll()
                .requestMatchers("/user/**").hasAnyRole("USER", "ADMIN")
                .requestMatchers("/admin/**").hasRole("ADMIN")
                .requestMatchers("/api/**").hasAuthority("API_ACCESS")
                .anyRequest().authenticated()
            );
            
        return http.build();
    }
}
```

### 4.2 方法级别的授权

```java
@Configuration
@EnableMethodSecurity
public class MethodSecurityConfig {
}
```

```java
@RestController
@RequestMapping("/api/orders")
public class OrderController {
    
    @GetMapping
    @PreAuthorize("hasRole('USER')")
    public List<Order> getAllOrders() {
        // 获取所有订单
        return orderService.findAll();
    }
    
    @GetMapping("/{id}")
    @PreAuthorize("hasRole('USER') and @orderSecurity.isOrderOwner(authentication, #id)")
    public Order getOrder(@PathVariable Long id) {
        // 获取特定订单
        return orderService.findById(id);
    }
    
    @PostMapping
    @PreAuthorize("hasRole('USER')")
    public Order createOrder(@RequestBody Order order) {
        // 创建订单
        return orderService.save(order);
    }
    
    @DeleteMapping("/{id}")
    @PreAuthorize("hasRole('ADMIN')")
    public void deleteOrder(@PathVariable Long id) {
        // 删除订单
        orderService.deleteById(id);
    }
}
```

```java
@Component("orderSecurity")
public class OrderSecurityEvaluator {
    
    @Autowired
    private OrderService orderService;
    
    public boolean isOrderOwner(Authentication authentication, Long orderId) {
        String username = authentication.getName();
        Order order = orderService.findById(orderId);
        return order != null && order.getUsername().equals(username);
    }
}
```

## 5. OAuth2和JWT集成

### 5.1 OAuth2资源服务器配置

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
</dependency>
```

```java
@Configuration
@EnableWebSecurity
public class ResourceServerConfig {
    
    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(authorize -> authorize
                .requestMatchers("/public/**").permitAll()
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt
                    .jwtAuthenticationConverter(jwtAuthenticationConverter())
                )
            );
            
        return http.build();
    }
    
    private Converter<Jwt, AbstractAuthenticationToken> jwtAuthenticationConverter() {
        JwtAuthenticationConverter converter = new JwtAuthenticationConverter();
        converter.setJwtGrantedAuthoritiesConverter(jwt -> {
            List<String> authorities = jwt.getClaimAsStringList("authorities");
            if (authorities == null) {
                authorities = Collections.emptyList();
            }
            return authorities.stream()
                .map(SimpleGrantedAuthority::new)
                .collect(Collectors.toList());
        });
        return converter;
    }
    
    @Bean
    public JwtDecoder jwtDecoder() {
        return NimbusJwtDecoder.withPublicKey(rsaPublicKey()).build();
    }
    
    @Bean
    public RSAPublicKey rsaPublicKey() {
        // 加载公钥
        try {
            Resource resource = new ClassPathResource("public.pem");
            String key = IOUtils.toString(resource.getInputStream(), StandardCharsets.UTF_8);
            key = key.replace("-----BEGIN PUBLIC KEY-----", "")
                     .replace("-----END PUBLIC KEY-----", "")
                     .replaceAll("\\s", "");
            byte[] decoded = Base64.getDecoder().decode(key);
            KeyFactory keyFactory = KeyFactory.getInstance("RSA");
            return (RSAPublicKey) keyFactory.generatePublic(new X509EncodedKeySpec(decoded));
        } catch (Exception e) {
            throw new RuntimeException("无法加载RSA公钥", e);
        }
    }
}
```

### 5.2 JWT认证过滤器实现

```java
@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {
    
    @Autowired
    private JwtTokenProvider tokenProvider;
    
    @Autowired
    private UserDetailsService userDetailsService;
    
    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, 
                                    FilterChain filterChain) throws ServletException, IOException {
        try {
            String jwt = getJwtFromRequest(request);
            
            if (StringUtils.hasText(jwt) && tokenProvider.validateToken(jwt)) {
                String username = tokenProvider.getUsernameFromJWT(jwt);
                UserDetails userDetails = userDetailsService.loadUserByUsername(username);
                
                UsernamePasswordAuthenticationToken authentication = 
                    new UsernamePasswordAuthenticationToken(userDetails, null, userDetails.getAuthorities());
                authentication.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
                
                SecurityContextHolder.getContext().setAuthentication(authentication);
            }
        } catch (Exception ex) {
            logger.error("无法设置用户认证", ex);
        }
        
        filterChain.doFilter(request, response);
    }
    
    private String getJwtFromRequest(HttpServletRequest request) {
        String bearerToken = request.getHeader("Authorization");
        if (StringUtils.hasText(bearerToken) && bearerToken.startsWith("Bearer ")) {
            return bearerToken.substring(7);
        }
        return null;
    }
}
```

## 6. 实践案例

### 6.1 记住我功能实现

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    
    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(authorize -> authorize
                .requestMatchers("/public/**").permitAll()
                .anyRequest().authenticated()
            )
            .formLogin(form -> form
                .loginPage("/login")
                .permitAll()
            )
            .rememberMe(remember -> remember
                .key("uniqueAndSecretKey")
                .tokenValiditySeconds(86400) // 1天
                .rememberMeParameter("remember-me")
            );
            
        return http.build();
    }
}
```

### 6.2 CSRF防护

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    
    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf
                .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
                .ignoringRequestMatchers("/api/webhook/**")
            );
            
        return http.build();
    }
}
```

```html
<form action="/process" method="post">
    <input type="hidden" name="${_csrf.parameterName}" value="${_csrf.token}"/>
    <!-- 表单字段 -->
    <button type="submit">提交</button>
</form>
```

## 7. 最佳实践

1. 认证安全
   - 使用强密码策略
   - 实现账户锁定机制
   - 采用多因素认证

2. 授权控制
   - 遵循最小权限原则
   - 实现细粒度的权限控制
   - 定期审计权限分配

3. 安全防护
   - 防止XSS攻击
   - 实施CSRF保护
   - 保护敏感数据

## 总结

通过本文的学习，