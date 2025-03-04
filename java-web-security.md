# Day 28: Java Web开发 - 安全与身份认证

## 引言

Web应用的安全性和身份认证是保护用户数据和系统资源的关键。本文将详细介绍Java Web开发中的安全机制和身份认证方案。

## 1. 基本安全概念

### 1.1 认证与授权

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
            .authorizeRequests()
                .antMatchers("/public/**").permitAll()
                .antMatchers("/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
                .and()
            .formLogin()
                .loginPage("/login")
                .permitAll()
                .and()
            .logout()
                .permitAll();
    }
    
    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.userDetailsService(userDetailsService)
            .passwordEncoder(passwordEncoder());
    }
    
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

### 1.2 用户认证服务

```java
@Service
public class UserDetailsServiceImpl implements UserDetailsService {
    
    @Autowired
    private UserRepository userRepository;
    
    @Override
    public UserDetails loadUserByUsername(String username) 
            throws UsernameNotFoundException {
        User user = userRepository.findByUsername(username)
            .orElseThrow(() -> new UsernameNotFoundException(
                "User not found: " + username));
        
        return new org.springframework.security.core.userdetails.User(
            user.getUsername(),
            user.getPassword(),
            getAuthorities(user.getRoles()));
    }
    
    private Collection<? extends GrantedAuthority> getAuthorities(Set<Role> roles) {
        return roles.stream()
            .map(role -> new SimpleGrantedAuthority(role.getName()))
            .collect(Collectors.toList());
    }
}
```

## 2. JWT认证

### 2.1 JWT配置

```java
@Configuration
public class JwtConfig {
    @Value("${jwt.secret}")
    private String secret;
    
    @Value("${jwt.expiration}")
    private Long expiration;
    
    @Bean
    public JwtTokenProvider jwtTokenProvider() {
        return new JwtTokenProvider(secret, expiration);
    }
}

public class JwtTokenProvider {
    private final String secret;
    private final Long expiration;
    
    public String generateToken(Authentication authentication) {
        UserDetails userDetails = (UserDetails) authentication.getPrincipal();
        Date now = new Date();
        Date expiryDate = new Date(now.getTime() + expiration);
        
        return Jwts.builder()
            .setSubject(userDetails.getUsername())
            .setIssuedAt(now)
            .setExpiration(expiryDate)
            .signWith(SignatureAlgorithm.HS512, secret)
            .compact();
    }
    
    public boolean validateToken(String token) {
        try {
            Jwts.parser().setSigningKey(secret).parseClaimsJws(token);
            return true;
        } catch (JwtException | IllegalArgumentException e) {
            return false;
        }
    }
}
```

### 2.2 JWT过滤器

```java
@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {
    
    @Autowired
    private JwtTokenProvider tokenProvider;
    
    @Autowired
    private UserDetailsService userDetailsService;
    
    @Override
    protected void doFilterInternal(HttpServletRequest request,
            HttpServletResponse response, FilterChain filterChain)
            throws ServletException, IOException {
        try {
            String jwt = getJwtFromRequest(request);
            
            if (StringUtils.hasText(jwt) && tokenProvider.validateToken(jwt)) {
                String username = tokenProvider.getUsernameFromJWT(jwt);
                UserDetails userDetails = userDetailsService.loadUserByUsername(username);
                UsernamePasswordAuthenticationToken authentication =
                    new UsernamePasswordAuthenticationToken(
                        userDetails, null, userDetails.getAuthorities());
                SecurityContextHolder.getContext().setAuthentication(authentication);
            }
        } catch (Exception ex) {
            logger.error("Could not set user authentication in security context", ex);
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

## 3. OAuth2认证

### 3.1 OAuth2配置

```java
@Configuration
@EnableOAuth2Client
public class OAuth2Config {
    
    @Value("${oauth2.client-id}")
    private String clientId;
    
    @Value("${oauth2.client-secret}")
    private String clientSecret;
    
    @Bean
    public OAuth2RestTemplate oauth2RestTemplate(
            OAuth2ClientContext oauth2ClientContext,
            OAuth2ProtectedResourceDetails details) {
        return new OAuth2RestTemplate(details, oauth2ClientContext);
    }
    
    @Bean
    @ConfigurationProperties("oauth2.client")
    public AuthorizationCodeResourceDetails oauth2Details() {
        return new AuthorizationCodeResourceDetails();
    }
}
```

### 3.2 OAuth2控制器

```java
@RestController
@RequestMapping("/auth")
public class OAuth2Controller {
    
    @Autowired
    private OAuth2RestTemplate oauth2RestTemplate;
    
    @GetMapping("/login")
    public ResponseEntity<String> login() {
        String url = oauth2RestTemplate.getResource().getUserAuthorizationUri();
        return ResponseEntity.ok(url);
    }
    
    @GetMapping("/callback")
    public ResponseEntity<UserInfo> callback(@RequestParam String code) {
        try {
            OAuth2AccessToken accessToken = oauth2RestTemplate.getAccessToken();
            UserInfo userInfo = oauth2RestTemplate.getForObject(
                "user-info-endpoint", UserInfo.class);
            return ResponseEntity.ok(userInfo);
        } catch (OAuth2Exception e) {
            return ResponseEntity.status(HttpStatus.UNAUTHORIZED).build();
        }
    }
}
```

## 4. 安全最佳实践

### 4.1 密码加密

```java
@Service
public class PasswordService {
    
    @Autowired
    private PasswordEncoder passwordEncoder;
    
    public String encodePassword(String rawPassword) {
        return passwordEncoder.encode(rawPassword);
    }
    
    public boolean matches(String rawPassword, String encodedPassword) {
        return passwordEncoder.matches(rawPassword, encodedPassword);
    }
}

@Service
public class UserService {
    
    @Autowired
    private PasswordService passwordService;
    
    @Autowired
    private UserRepository userRepository;
    
    public User createUser(UserRegistrationDto dto) {
        User user = new User();
        user.setUsername(dto.getUsername());
        user.setPassword(passwordService.encodePassword(dto.getPassword()));
        return userRepository.save(user);
    }
}
```

### 4.2 CSRF防护

```java
@Configuration
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
            .csrf()
                .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
                .and()
            .addFilterAfter(
                new CsrfHeaderFilter(), 
                CsrfFilter.class);
    }
}

public class CsrfHeaderFilter extends OncePerRequestFilter {
    
    @Override
    protected void doFilterInternal(HttpServletRequest request,
            HttpServletResponse response, FilterChain filterChain)
            throws ServletException, IOException {
        CsrfToken csrf = (CsrfToken) request.getAttribute(CsrfToken.class.getName());
        if (csrf != null) {
            Cookie cookie = WebUtils.getCookie(request, "XSRF-TOKEN");
            String token = csrf.getToken();
            if (cookie == null || token != null && !token.equals(cookie.getValue())) {
                cookie = new Cookie("XSRF-TOKEN", token);
                cookie.setPath("/");
                response.addCookie(cookie);
            }
        }
        filterChain.doFilter(request, response);
    }
}
```

## 5. 实践案例

### 5.1 登录认证

```java
@RestController
@RequestMapping("/api/auth")
public class AuthController {
    
    @Autowired
    private AuthenticationManager authenticationManager;
    
    @Autowired
    private JwtTokenProvider tokenProvider;
    
    @PostMapping("/login")
    public ResponseEntity<?> login(@RequestBody LoginRequest loginRequest) {
        try {
            Authentication authentication = authenticationManager.authenticate(
                new UsernamePasswordAuthenticationToken(
                    loginRequest.getUsername(),
                    loginRequest.getPassword()
                )
            );
            
            SecurityContextHolder.getContext().setAuthentication(authentication);
            String jwt = tokenProvider.generateToken(authentication);
            
            return ResponseEntity.ok(new JwtAuthenticationResponse(jwt));
        } catch (AuthenticationException e) {
            return ResponseEntity
                .status(HttpStatus.UNAUTHORIZED)
                .body(new ApiResponse(false, "Invalid username or password"));
        }
    }
}
```

### 5.2 角色权限控制

```java
@PreAuthorize("hasRole('ADMIN')")
@RestController
@RequestMapping("/api/admin")
public class AdminController {
    
    @GetMapping("/users")
    public ResponseEntity<List<User>> getAllUsers() {
        // 只有ADMIN角色可以访问
        return ResponseEntity.ok(userService.findAll());
    }
    
    @PostMapping("/users/{id}/role")
    public ResponseEntity<?> updateUserRole(
            @PathVariable Long id,
            @RequestBody RoleRequest roleRequest) {
        userService.updateRole(id, roleRequest.getRole());
        return ResponseEntity.ok().build();
    }
}

@Service
public class UserService {
    
    @PreAuthorize("hasRole('ADMIN')")
    public void updateRole(Long userId, String roleName) {
        User user = findById(userId);
        Role role = roleRepository.findByName(roleName)
            .orElseThrow(() -> new RoleNotFoundException(roleName));
        
        user.getRoles().add(role);
        userRepository.save(user);
    }
}
```

## 6. 最佳实践

1. 使用HTTPS保护传输数据
2. 实现适当的密码策略
3. 防范常见的Web安全漏洞
4. 实现请求限流和防刷机制
5. 定期更新安全依赖

## 总结

本文介绍了Java Web开发中的安全与身份认证机制，包括：

1. 基本的安全概念
2. JWT认证实现
3. OAuth2集成
4. 安全最佳实践
5. 实践案例

通过掌握这些知识，我们可以构建更安全的Web应用程序，有效保护用户数据和系统资源。

## 参考资源

1. Spring Security文档
2. JWT规范和最佳实践
3. OAuth 2.0规范
4. OWASP安全指南