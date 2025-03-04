# Day 19: Java Spring框架 - MVC

## 引言

Spring MVC是Spring框架中的Web模块，它实现了Model-View-Controller设计模式，为开发Web应用提供了强大而灵活的支持。本文将详细介绍Spring MVC的核心概念和实践应用。

## 1. Spring MVC基础

### 1.1 MVC架构

- Model：数据模型，包含业务逻辑和数据访问
- View：视图层，负责数据的展示
- Controller：控制器，处理用户请求并协调Model和View

### 1.2 DispatcherServlet

```xml
<!-- web.xml配置 -->
<servlet>
    <servlet-name>dispatcher</servlet-name>
    <servlet-class>
        org.springframework.web.servlet.DispatcherServlet
    </servlet-class>
    <init-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>/WEB-INF/spring-mvc.xml</param-value>
    </init-param>
    <load-on-startup>1</load-on-startup>
</servlet>

<servlet-mapping>
    <servlet-name>dispatcher</servlet-name>
    <url-pattern>/</url-pattern>
</servlet-mapping>
```

## 2. 控制器

### 2.1 基本控制器

```java
@Controller
@RequestMapping("/users")
public class UserController {
    @Autowired
    private UserService userService;
    
    @GetMapping
    public String listUsers(Model model) {
        model.addAttribute("users", userService.findAll());
        return "users/list";
    }
    
    @GetMapping("/{id}")
    public String getUser(@PathVariable Long id, Model model) {
        model.addAttribute("user", userService.findById(id));
        return "users/detail";
    }
    
    @PostMapping
    public String createUser(@ModelAttribute User user) {
        userService.save(user);
        return "redirect:/users";
    }
}
```

### 2.2 REST控制器

```java
@RestController
@RequestMapping("/api/users")
public class UserRestController {
    @Autowired
    private UserService userService;
    
    @GetMapping
    public List<User> getUsers() {
        return userService.findAll();
    }
    
    @GetMapping("/{id}")
    public ResponseEntity<User> getUser(@PathVariable Long id) {
        User user = userService.findById(id);
        if (user == null) {
            return ResponseEntity.notFound().build();
        }
        return ResponseEntity.ok(user);
    }
    
    @PostMapping
    public ResponseEntity<User> createUser(@RequestBody User user) {
        User savedUser = userService.save(user);
        return ResponseEntity
            .created(URI.create("/api/users/" + savedUser.getId()))
            .body(savedUser);
    }
    
    @PutMapping("/{id}")
    public ResponseEntity<User> updateUser(
            @PathVariable Long id, 
            @RequestBody User user) {
        user.setId(id);
        User updatedUser = userService.update(user);
        return ResponseEntity.ok(updatedUser);
    }
    
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteUser(@PathVariable Long id) {
        userService.delete(id);
        return ResponseEntity.noContent().build();
    }
}
```

## 3. 视图解析

### 3.1 配置视图解析器

```java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {
    @Bean
    public ViewResolver viewResolver() {
        InternalResourceViewResolver resolver = 
            new InternalResourceViewResolver();
        resolver.setPrefix("/WEB-INF/views/");
        resolver.setSuffix(".jsp");
        return resolver;
    }
}
```

### 3.2 Thymeleaf配置

```java
@Configuration
@EnableWebMvc
public class ThymeleafConfig {
    @Bean
    public SpringTemplateEngine templateEngine() {
        SpringTemplateEngine engine = new SpringTemplateEngine();
        engine.setTemplateResolver(templateResolver());
        return engine;
    }
    
    @Bean
    public SpringResourceTemplateResolver templateResolver() {
        SpringResourceTemplateResolver resolver = 
            new SpringResourceTemplateResolver();
        resolver.setPrefix("/WEB-INF/templates/");
        resolver.setSuffix(".html");
        resolver.setTemplateMode(TemplateMode.HTML);
        return resolver;
    }
}
```

## 4. 数据绑定和验证

### 4.1 表单绑定

```java
@Controller
@RequestMapping("/register")
public class RegistrationController {
    @GetMapping
    public String showForm(Model model) {
        model.addAttribute("user", new User());
        return "registration/form";
    }
    
    @PostMapping
    public String processForm(
            @Valid @ModelAttribute("user") User user,
            BindingResult result) {
        if (result.hasErrors()) {
            return "registration/form";
        }
        // 处理注册逻辑
        return "redirect:/login";
    }
}
```

### 4.2 数据验证

```java
public class User {
    @NotBlank(message = "用户名不能为空")
    private String username;
    
    @Email(message = "邮箱格式不正确")
    private String email;
    
    @Size(min = 6, message = "密码至少6个字符")
    private String password;
    
    // getters and setters
}

@Controller
public class ValidationController {
    @InitBinder
    public void initBinder(WebDataBinder binder) {
        binder.addValidators(new UserValidator());
    }
    
    @PostMapping("/validate")
    public String validateUser(
            @Valid @ModelAttribute User user,
            BindingResult result) {
        if (result.hasErrors()) {
            // 处理验证错误
            return "form";
        }
        return "success";
    }
}
```

## 5. 文件上传

```java
@Controller
public class FileUploadController {
    @PostMapping("/upload")
    public String handleFileUpload(
            @RequestParam("file") MultipartFile file,
            RedirectAttributes redirectAttributes) {
        if (file.isEmpty()) {
            redirectAttributes.addFlashAttribute(
                "message", "请选择文件");
            return "redirect:/uploadForm";
        }
        
        try {
            byte[] bytes = file.getBytes();
            Path path = Paths.get("uploads/" + 
                file.getOriginalFilename());
            Files.write(path, bytes);
            
            redirectAttributes.addFlashAttribute(
                "message", "文件上传成功");
        } catch (IOException e) {
            e.printStackTrace();
        }
        
        return "redirect:/uploadForm";
    }
}
```

## 6. 异常处理

```java
@ControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(UserNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleUserNotFound(
            UserNotFoundException ex) {
        ErrorResponse error = new ErrorResponse(
            HttpStatus.NOT_FOUND.value(),
            ex.getMessage());
        return new ResponseEntity<>(error, HttpStatus.NOT_FOUND);
    }
    
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGlobalException(
            Exception ex) {
        ErrorResponse error = new ErrorResponse(
            HttpStatus.INTERNAL_SERVER_ERROR.value(),
            "服务器内部错误");
        return new ResponseEntity<>(error, 
            HttpStatus.INTERNAL_SERVER_ERROR);
    }
}

@ResponseStatus(HttpStatus.NOT_FOUND)
public class UserNotFoundException extends RuntimeException {
    public UserNotFoundException(String message) {
        super(message);
    }
}
```

## 7. 拦截器

```java
public class AuthenticationInterceptor 
        implements HandlerInterceptor {
    
    @Override
    public boolean preHandle(
            HttpServletRequest request,
            HttpServletResponse response,
            Object handler) throws Exception {
        HttpSession session = request.getSession();
        if (session.getAttribute("user") == null) {
            response.sendRedirect("/login");
            return false;
        }
        return true;
    }
}

@Configuration
public class WebMvcConfig implements WebMvcConfigurer {
    @Override
    public void addInterceptors(
            InterceptorRegistry registry) {
        registry.addInterceptor(new AuthenticationInterceptor())
            .addPathPatterns("/secure/**")
            .excludePathPatterns("/login", "/register");
    }
}
```

## 8. 最佳实践

1. 使用适当的HTTP方法
2. 实现合理的URL设计
3. 正确处理异常
4. 使用数据验证
5. 实现适当的安全措施

## 总结

本文介绍了Spring MVC的核心概念和实践应用，包括：

1. MVC架构和基本配置
2. 控制器的实现
3. 视图解析器的配置
4. 数据绑定和验证
5. 文件上传处理
6. 异常处理机制
7. 拦截器的使用

通过掌握这些知识，我们可以使用Spring MVC开发出结构清晰、功能完善的Web应用。

## 参考资源

1. Spring MVC官方文档
2. Spring实战（第5版）
3. RESTful Web Services最佳实践
4. Spring MVC测试指南