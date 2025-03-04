# Day 26: Java Web开发 - MVC框架

## 引言

MVC（Model-View-Controller）框架是现代Web应用开发中最常用的架构模式之一。本文将详细介绍Java Web MVC框架的核心概念和实践应用。

## 1. MVC基础

### 1.1 MVC架构

- Model（模型）：处理业务逻辑和数据访问
- View（视图）：负责数据的展示
- Controller（控制器）：处理用户请求，协调Model和View

### 1.2 基本流程

```java
// Controller
@Controller
@RequestMapping("/users")
public class UserController {
    @Autowired
    private UserService userService;
    
    @GetMapping
    public String listUsers(Model model) {
        List<User> users = userService.findAll();
        model.addAttribute("users", users);
        return "users/list";
    }
    
    @GetMapping("/{id}")
    public String getUser(@PathVariable Long id, Model model) {
        User user = userService.findById(id);
        model.addAttribute("user", user);
        return "users/detail";
    }
}

// Model
@Service
public class UserService {
    @Autowired
    private UserRepository userRepository;
    
    public List<User> findAll() {
        return userRepository.findAll();
    }
    
    public User findById(Long id) {
        return userRepository.findById(id)
            .orElseThrow(() -> new UserNotFoundException(id));
    }
}

// View (users/list.jsp)
<%@ page language="java" contentType="text/html; charset=UTF-8"
    pageEncoding="UTF-8"%>
<%@ taglib prefix="c" uri="http://java.sun.com/jsp/jstl/core"%>
<!DOCTYPE html>
<html>
<head>
    <title>用户列表</title>
</head>
<body>
    <h1>用户列表</h1>
    <table>
        <tr>
            <th>ID</th>
            <th>用户名</th>
            <th>邮箱</th>
        </tr>
        <c:forEach items="${users}" var="user">
            <tr>
                <td>${user.id}</td>
                <td>${user.username}</td>
                <td>${user.email}</td>
            </tr>
        </c:forEach>
    </table>
</body>
</html>
```

## 2. 请求处理

### 2.1 请求映射

```java
@Controller
@RequestMapping("/api")
public class ApiController {
    @GetMapping("/data")
    @ResponseBody
    public Map<String, Object> getData() {
        Map<String, Object> data = new HashMap<>();
        data.put("message", "Hello, World!");
        return data;
    }
    
    @PostMapping("/submit")
    public String submitForm(@ModelAttribute FormData form) {
        // 处理表单数据
        return "redirect:/success";
    }
    
    @RequestMapping(value = "/custom", method = {RequestMethod.GET, RequestMethod.POST})
    public void handleCustom(HttpServletRequest request,
            HttpServletResponse response) {
        // 自定义请求处理
    }
}
```

### 2.2 参数绑定

```java
@Controller
public class ParameterBindingController {
    @GetMapping("/path/{id}")
    public String pathVariable(@PathVariable("id") Long id) {
        return "view";
    }
    
    @GetMapping("/query")
    public String requestParam(
            @RequestParam("name") String name,
            @RequestParam(defaultValue = "1") int page) {
        return "view";
    }
    
    @PostMapping("/form")
    public String formData(@ModelAttribute User user) {
        return "view";
    }
    
    @PostMapping("/json")
    @ResponseBody
    public ResponseEntity<String> jsonData(@RequestBody User user) {
        return ResponseEntity.ok("Success");
    }
}
```

## 3. 视图解析

### 3.1 视图解析器配置

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
    
    @Bean
    public MessageSource messageSource() {
        ResourceBundleMessageSource source = 
            new ResourceBundleMessageSource();
        source.setBasename("messages");
        return source;
    }
}
```

### 3.2 视图技术

```java
// Thymeleaf配置
@Configuration
public class ThymeleafConfig {
    @Bean
    public SpringTemplateEngine templateEngine() {
        SpringTemplateEngine engine = new SpringTemplateEngine();
        engine.setTemplateResolver(templateResolver());
        return engine;
    }
    
    @Bean
    public ITemplateResolver templateResolver() {
        SpringResourceTemplateResolver resolver = 
            new SpringResourceTemplateResolver();
        resolver.setPrefix("/WEB-INF/templates/");
        resolver.setSuffix(".html");
        resolver.setTemplateMode(TemplateMode.HTML);
        return resolver;
    }
}
```

## 4. 数据验证

### 4.1 表单验证

```java
@Data
public class UserForm {
    @NotBlank(message = "用户名不能为空")
    @Size(min = 4, max = 20, message = "用户名长度必须在4-20之间")
    private String username;
    
    @Email(message = "邮箱格式不正确")
    private String email;
    
    @NotNull(message = "年龄不能为空")
    @Min(value = 18, message = "年龄必须大于等于18")
    private Integer age;
}

@Controller
public class ValidationController {
    @PostMapping("/register")
    public String register(@Valid @ModelAttribute UserForm form,
            BindingResult result, Model model) {
        if (result.hasErrors()) {
            return "register";
        }
        // 处理注册逻辑
        return "redirect:/success";
    }
}
```

## 5. 异常处理

### 5.1 全局异常处理

```java
@ControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(UserNotFoundException.class)
    public ModelAndView handleUserNotFound(UserNotFoundException ex) {
        ModelAndView mav = new ModelAndView("error/404");
        mav.addObject("message", ex.getMessage());
        return mav;
    }
    
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGlobalException(Exception ex) {
        ErrorResponse error = new ErrorResponse(
            HttpStatus.INTERNAL_SERVER_ERROR.value(),
            "服务器内部错误");
        return new ResponseEntity<>(error, HttpStatus.INTERNAL_SERVER_ERROR);
    }
}
```

## 6. 实践案例

### 6.1 文件上传

```java
@Controller
public class FileUploadController {
    @Value("${file.upload.dir}")
    private String uploadDir;
    
    @PostMapping("/upload")
    public String handleFileUpload(
            @RequestParam("file") MultipartFile file,
            RedirectAttributes redirectAttributes) {
        if (file.isEmpty()) {
            redirectAttributes.addFlashAttribute(
                "error", "请选择文件");
            return "redirect:/uploadForm";
        }
        
        try {
            String fileName = file.getOriginalFilename();
            Path path = Paths.get(uploadDir + fileName);
            Files.copy(file.getInputStream(), path,
                StandardCopyOption.REPLACE_EXISTING);
            
            redirectAttributes.addFlashAttribute(
                "message", "文件上传成功");
        } catch (IOException e) {
            e.printStackTrace();
            redirectAttributes.addFlashAttribute(
                "error", "文件上传失败");
        }
        
        return "redirect:/uploadForm";
    }
}
```

### 6.2 RESTful API

```java
@RestController
@RequestMapping("/api/users")
public class UserApiController {
    @Autowired
    private UserService userService;
    
    @GetMapping
    public List<User> getUsers() {
        return userService.findAll();
    }
    
    @GetMapping("/{id}")
    public ResponseEntity<User> getUser(@PathVariable Long id) {
        return userService.findById(id)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }
    
    @PostMapping
    public ResponseEntity<User> createUser(@RequestBody User user) {
        User savedUser = userService.save(user);
        return ResponseEntity
            .created(URI.create("/api/users/" + savedUser.getId()))
            .body(savedUser);
    }
}
```

## 7. 最佳实践

1. 遵循RESTful API设计原则
2. 使用适当的HTTP方法
3. 实现合理的错误处理
4. 注意安全性考虑
5. 保持代码整洁和可维护

## 总结

本文介绍了Java Web MVC框架的核心概念和实践应用，包括：

1. MVC架构的基本概念
2. 请求处理和参数绑定
3. 视图解析和渲染
4. 数据验证
5. 异常处理
6. 实践案例

通过掌握这些知识，我们可以开发出结构清晰、易于维护的Web应用程序。

## 参考资源

1. Spring MVC官方文档
2. RESTful Web Services最佳实践
3. Java Web开发实战
4. MVC设计模式指南