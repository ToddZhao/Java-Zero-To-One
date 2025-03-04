# Day 98: Java安全加固 - 企业级应用安全防护

## 1. 引言

在当今网络安全威胁日益增长的环境中，Java应用的安全加固变得尤为重要。本文将深入探讨Java企业级应用的安全加固技术、最佳实践和防护措施，帮助开发者构建更安全的系统。

## 2. Java安全基础

### 2.1 常见安全威胁

企业级Java应用面临的主要安全威胁包括：

- 注入攻击（SQL注入、命令注入等）
- 跨站脚本攻击（XSS）
- 跨站请求伪造（CSRF）
- 身份认证与会话管理缺陷
- 敏感数据泄露
- 权限控制缺陷
- 依赖组件漏洞
- 反序列化漏洞

### 2.2 安全设计原则

- 纵深防御：构建多层次的安全防护
- 最小权限原则：只授予必要的最小权限
- 安全默认配置：默认配置应当是安全的
- 数据保密性：保护敏感数据不被未授权访问
- 失败安全：系统在失败时应保持安全状态
- 完整性校验：验证数据的完整性和真实性

## 3. 代码级安全加固

### 3.1 输入验证与过滤

```java
public class InputValidator {
    // 使用正则表达式验证输入
    public static boolean isValidUsername(String username) {
        // 只允许字母、数字和下划线，长度5-20
        return username != null && username.matches("^[a-zA-Z0-9_]{5,20}$");
    }
    
    // 防止SQL注入的参数验证
    public static boolean isValidOrderBy(String orderBy) {
        // 只允许字母、数字和下划线，以及空格和逗号
        return orderBy != null && orderBy.matches("^[a-zA-Z0-9_, ]+$");
    }
    
    // HTML内容过滤，防止XSS
    public static String sanitizeHtml(String html) {
        PolicyFactory policy = Sanitizers.FORMATTING.and(Sanitizers.BLOCKS)
                .and(Sanitizers.LINKS);
        return policy.sanitize(html);
    }
}
```

### 3.2 SQL注入防护

```java
@Repository
public class SecureUserRepository {
    @PersistenceContext
    private EntityManager entityManager;
    
    // 不安全的查询方式
    public List<User> findUsersByRoleUnsafe(String role) {
        // 危险：直接拼接SQL字符串
        String query = "SELECT * FROM users WHERE role = '" + role + "'";
        return entityManager.createNativeQuery(query, User.class).getResultList();
    }
    
    // 安全的查询方式：使用参数化查询
    public List<User> findUsersByRoleSafe(String role) {
        return entityManager.createQuery("SELECT u FROM User u WHERE u.role = :role", User.class)
                .setParameter("role", role)
                .getResultList();
    }
    
    // 使用命名查询进一步提高安全性
    @SuppressWarnings("unchecked")
    public List<User> findUsersByRoleNamedQuery(String role) {
        return entityManager.createNamedQuery("User.findByRole")
                .setParameter("role", role)
                .getResultList();
    }
}
```

### 3.3 XSS防护

```java
@RestController
public class SecureController {
    // 不安全的响应处理
    @GetMapping("/unsafe/message")
    public String getUnsafeMessage(@RequestParam String message) {
        // 危险：直接返回用户输入
        return "<div>" + message + "</div>";
    }
    
    // 安全的响应处理：转义HTML
    @GetMapping("/safe/message")
    public String getSafeMessage(@RequestParam String message) {
        // 使用Spring的HtmlUtils转义HTML
        return "<div>" + HtmlUtils.htmlEscape(message) + "</div>";
    }
    
    // 使用Content-Security-Policy头
    @GetMapping("/secure/page")
    public ResponseEntity<String> getSecurePage() {
        String html = "<html><body><h1>Secure Page</h1></body></html>";
        return ResponseEntity.ok()
                .header("Content-Security-Policy", "default-src 'self'; script-src 'self'")
                .body(html);
    }
}
```

## 4. 认证与授权加固

### 4.1 强密码策略

```java
@Component
public class PasswordValidator {
    // 密码强度验证
    public boolean isStrongPassword(String password) {
        // 至少8位，包含大小写字母、数字和特殊字符
        String pattern = "^(?=.*[0-9])(?=.*[a-z])(?=.*[A-Z])(?=.*[@#$%^&+=])(?=\\S+$).{8,}$";
        return password != null && password.matches(pattern);
    }
    
    // 密码哈希
    public String hashPassword(String password) {
        return BCrypt.hashpw(password, BCrypt.gensalt(12));
    }
    
    // 密码验证
    public boolean verifyPassword(String password, String hashedPassword) {
        return BCrypt.checkpw(password, hashedPassword);
    }
}
```

### 4.2 JWT安全配置

```java
@Configuration
public class JwtSecurityConfig {
    @Value("${jwt.secret}")
    private String secret;
    
    @Value("${jwt.expiration}")
    private long expiration;
    
    @Bean
    public JwtTokenProvider jwtTokenProvider() {
        return new JwtTokenProvider(secret, expiration);
    }
}

public class JwtTokenProvider {
    private final String secret;
    private final long expiration;
    
    public JwtTokenProvider(String secret, long expiration) {
        this.secret = secret;
        this.expiration = expiration;
    }
    
    // 生成JWT令牌
    public String generateToken(UserDetails userDetails) {
        Map<String, Object> claims = new HashMap<>();
        Collection<? extends GrantedAuthority> authorities = userDetails.getAuthorities();
        claims.put("authorities", authorities.stream()
                .map(GrantedAuthority::getAuthority)
                .collect(Collectors.toList()));
        
        return Jwts.builder()
                .setClaims(claims)
                .setSubject(userDetails.getUsername())
                .setIssuedAt(new Date())
                .setExpiration(new Date(System.currentTimeMillis() + expiration))
                .signWith(SignatureAlgorithm.HS512, secret)
                .compact();
    }
    
    // 验证JWT令牌
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

### 4.3 OAuth2安全配置

```java
@Configuration
@EnableWebSecurity
@EnableGlobalMethodSecurity(prePostEnabled = true)
public class OAuth2SecurityConfig extends WebSecurityConfigurerAdapter {
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
            .csrf().disable()
            .authorizeRequests()
                .antMatchers("/api/public/**").permitAll()
                .antMatchers("/api/user/**").hasRole("USER")
                .antMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            .and()
            .oauth2Login()
                .loginPage("/login")
                .defaultSuccessUrl("/home")
                .failureUrl("/login?error=true")
            .and()
            .oauth2ResourceServer()
                .jwt();
    }
    
    @Bean
    public JwtDecoder jwtDecoder() {
        return JwtDecoders.fromIssuerLocation("https://your-auth-server.com");
    }
}
```

## 5. 数据安全加固

### 5.1 敏感数据加密

```java
@Service
public class DataEncryptionService {
    @Value("${encryption.key}")
    private String encryptionKey;
    
    private SecretKey secretKey;
    
    @PostConstruct
    public void init() {
        // 从Base64编码的字符串初始化密钥
        byte[] decodedKey = Base64.getDecoder().decode(encryptionKey);
        secretKey = new SecretKeySpec(decodedKey, 0, decodedKey.length, "AES");
    }
    
    // 加密数据
    public String encrypt(String data) throws Exception {
        Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");
        byte[] iv = new byte[12];
        new SecureRandom().nextBytes(iv);
        GCMParameterSpec parameterSpec = new GCMParameterSpec(128, iv);
        
        cipher.init(Cipher.ENCRYPT_MODE, secretKey, parameterSpec);
        byte[] encryptedData = cipher.doFinal(data.getBytes(StandardCharsets.UTF_8));
        
        // 将IV和加密数据合并
        ByteBuffer byteBuffer = ByteBuffer.allocate(iv.length + encryptedData.length);
        byteBuffer.put(iv);
        byteBuffer.put(encryptedData);
        
        return Base64.getEncoder().encodeToString(byteBuffer.array());
    }
    
    // 解密数据
    public String decrypt(String encryptedData) throws Exception {
        byte[] decodedData = Base64.getDecoder().decode(encryptedData);
        
        // 提取IV和加密数据
        ByteBuffer byteBuffer = ByteBuffer.wrap(decodedData);
        byte[] iv = new byte[12];
        byteBuffer.get(iv);
        byte[] cipherText = new byte[byteBuffer.remaining()];
        byteBuffer.get(cipherText);
        
        Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");
        GCMParameterSpec parameterSpec = new GCMParameterSpec(128, iv);
        cipher.init(Cipher.DECRYPT_MODE, secretKey, parameterSpec);
        
        byte[] decryptedData = cipher.doFinal(cipherText);
        return new String(decryptedData, StandardCharsets.UTF_8);
    }
}
```

### 5.2 安全的文件上传处理

```java
@Service
public class SecureFileUploadService {
    private static final List<String> ALLOWED_EXTENSIONS = Arrays.asList("jpg", "jpeg", "png", "pdf", "doc", "docx");
    private static final long MAX_FILE_SIZE = 10 * 1024 * 1024; // 10MB
    
    @Value("${upload.directory}")
    private String uploadDirectory;
    
    public String uploadFile(MultipartFile file) throws IOException, SecurityException {
        // 验证文件扩展名
        String originalFilename = file.getOriginalFilename();
        String extension = FilenameUtils.getExtension(originalFilename).toLowerCase();
        
        if (!ALLOWED_EXTENSIONS.contains(extension)) {
            throw new SecurityException("不允许的文件类型: " + extension);
        }
        
        // 验证文件大小
        if (file.getSize() > MAX_FILE_SIZE) {
            throw new SecurityException("文件大小超过限制");
        }
        
        // 生成安全的文件名
        String safeFilename = UUID.randomUUID().toString() + "." + extension;
        Path targetPath = Paths.get(uploadDirectory, safeFilename);
        
        // 保存文件
        Files.copy(file.getInputStream(), targetPath, StandardCopyOption.REPLACE_EXISTING);
        
        // 对于图片文件，可以进行额外的安全处理
        if (Arrays.asList("jpg", "jpeg", "png").contains(extension)) {
            sanitizeImage(targetPath.toFile());
        }
        
        return safeFilename;
    }
    
    private void sanitizeImage(File imageFile) {
        // 使用图像库重新处理图像，去除可能的恶意数据
        try {
            BufferedImage originalImage = ImageIO.read(imageFile);
            BufferedImage sanitizedImage = new BufferedImage(
                    originalImage.getWidth(), 
                    originalImage.getHeight(), 
                    originalImage.getType());
            
            Graphics2D g2d = sanitizedImage.createGraphics();
            g2d.drawImage(originalImage, 0, 0, null);
            g2d.dispose();
            
            ImageIO.write(sanitizedImage, Fil