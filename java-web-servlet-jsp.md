# Day 25: Java Web开发 - Servlet和JSP

## 引言

Servlet和JSP是Java Web开发的基础技术，它们为开发动态Web应用提供了强大的支持。本文将详细介绍Servlet和JSP的核心概念和实践应用。

## 1. Servlet基础

### 1.1 Servlet生命周期

```java
public class LifecycleServlet extends HttpServlet {
    @Override
    public void init() throws ServletException {
        System.out.println("Servlet初始化");
    }
    
    @Override
    protected void doGet(HttpServletRequest request, 
            HttpServletResponse response) throws ServletException, IOException {
        System.out.println("处理GET请求");
        response.setContentType("text/html;charset=UTF-8");
        PrintWriter out = response.getWriter();
        out.println("<html><body>");
        out.println("<h1>Hello, Servlet!</h1>");
        out.println("</body></html>");
    }
    
    @Override
    public void destroy() {
        System.out.println("Servlet销毁");
    }
}
```

### 1.2 Servlet配置

```xml
<!-- web.xml -->
<web-app>
    <servlet>
        <servlet-name>lifecycleServlet</servlet-name>
        <servlet-class>com.example.LifecycleServlet</servlet-class>
        <load-on-startup>1</load-on-startup>
    </servlet>
    
    <servlet-mapping>
        <servlet-name>lifecycleServlet</servlet-name>
        <url-pattern>/lifecycle</url-pattern>
    </servlet-mapping>
</web-app>
```

## 2. 请求和响应

### 2.1 处理请求

```java
@WebServlet("/user")
public class UserServlet extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest request, 
            HttpServletResponse response) throws ServletException, IOException {
        // 获取请求参数
        String name = request.getParameter("name");
        String age = request.getParameter("age");
        
        // 设置响应内容类型
        response.setContentType("text/html;charset=UTF-8");
        
        // 输出响应
        PrintWriter out = response.getWriter();
        out.println("<html><body>");
        out.println("<h2>用户信息</h2>");
        out.println("<p>姓名: " + name + "</p>");
        out.println("<p>年龄: " + age + "</p>");
        out.println("</body></html>");
    }
    
    @Override
    protected void doPost(HttpServletRequest request, 
            HttpServletResponse response) throws ServletException, IOException {
        // 设置请求编码
        request.setCharacterEncoding("UTF-8");
        
        // 处理POST请求
        String username = request.getParameter("username");
        String password = request.getParameter("password");
        
        // 重定向到其他页面
        response.sendRedirect("/success.jsp");
    }
}
```

### 2.2 Session管理

```java
@WebServlet("/session")
public class SessionServlet extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest request, 
            HttpServletResponse response) throws ServletException, IOException {
        // 获取或创建Session
        HttpSession session = request.getSession();
        
        // 设置Session属性
        session.setAttribute("user", "John");
        
        // 获取Session属性
        String user = (String) session.getAttribute("user");
        
        // 设置Session超时时间（分钟）
        session.setMaxInactiveInterval(30);
        
        // 使Session失效
        session.invalidate();
    }
}
```

## 3. JSP基础

### 3.1 JSP语法

```jsp
<%@ page language="java" contentType="text/html; charset=UTF-8"
    pageEncoding="UTF-8"%>
<%@ taglib prefix="c" uri="http://java.sun.com/jsp/jstl/core"%>
<!DOCTYPE html>
<html>
<head>
<title>JSP示例</title>
</head>
<body>
    <%-- JSP注释 --%>
    
    <%-- 声明 --%>
    <%! int count = 0; %>
    
    <%-- 脚本片段 --%>
    <% 
        count++;
        String message = "Hello, JSP!";
    %>
    
    <%-- 表达式 --%>
    <h1><%= message %></h1>
    <p>访问次数：<%= count %></p>
    
    <%-- JSTL标签 --%>
    <c:if test="${not empty param.name}">
        <p>欢迎, ${param.name}!</p>
    </c:if>
    
    <c:forEach var="i" begin="1" end="5">
        <p>Item ${i}</p>
    </c:forEach>
</body>
</html>
```

### 3.2 内置对象

```jsp
<%@ page language="java" contentType="text/html; charset=UTF-8"
    pageEncoding="UTF-8"%>
<!DOCTYPE html>
<html>
<head>
<title>JSP内置对象</title>
</head>
<body>
    <%-- request对象 --%>
    <p>请求方法：<%= request.getMethod() %></p>
    <p>客户端IP：<%= request.getRemoteAddr() %></p>
    
    <%-- response对象 --%>
    <% response.setHeader("Cache-Control", "no-cache"); %>
    
    <%-- session对象 --%>
    <% 
        session.setAttribute("user", "John");
        String user = (String) session.getAttribute("user");
    %>
    <p>用户：<%= user %></p>
    
    <%-- application对象 --%>
    <% 
        application.setAttribute("count", 100);
        Integer count = (Integer) application.getAttribute("count");
    %>
    <p>计数：<%= count %></p>
    
    <%-- out对象 --%>
    <% out.println("使用out对象输出"); %>
</body>
</html>
```

## 4. MVC模式

### 4.1 实现MVC架构

```java
// Model
public class User {
    private String username;
    private String email;
    
    // getters and setters
}

// Controller
@WebServlet("/users/*")
public class UserController extends HttpServlet {
    private UserService userService = new UserService();
    
    @Override
    protected void doGet(HttpServletRequest request, 
            HttpServletResponse response) throws ServletException, IOException {
        String action = request.getPathInfo();
        
        switch (action) {
            case "/list":
                listUsers(request, response);
                break;
            case "/edit":
                editUser(request, response);
                break;
            default:
                response.sendError(HttpServletResponse.SC_NOT_FOUND);
        }
    }
    
    private void listUsers(HttpServletRequest request, 
            HttpServletResponse response) throws ServletException, IOException {
        List<User> users = userService.getAllUsers();
        request.setAttribute("users", users);
        request.getRequestDispatcher("/WEB-INF/views/users.jsp")
            .forward(request, response);
    }
}

// View (users.jsp)
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
            <th>用户名</th>
            <th>邮箱</th>
        </tr>
        <c:forEach items="${users}" var="user">
            <tr>
                <td>${user.username}</td>
                <td>${user.email}</td>
            </tr>
        </c:forEach>
    </table>
</body>
</html>
```

## 5. 过滤器和监听器

### 5.1 过滤器

```java
@WebFilter("/*")
public class EncodingFilter implements Filter {
    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
        // 初始化
    }
    
    @Override
    public void doFilter(ServletRequest request, ServletResponse response,
            FilterChain chain) throws IOException, ServletException {
        request.setCharacterEncoding("UTF-8");
        response.setCharacterEncoding("UTF-8");
        chain.doFilter(request, response);
    }
    
    @Override
    public void destroy() {
        // 清理资源
    }
}
```

### 5.2 监听器

```java
@WebListener
public class SessionListener implements HttpSessionListener {
    @Override
    public void sessionCreated(HttpSessionEvent se) {
        System.out.println("Session created: " + se.getSession().getId());
    }
    
    @Override
    public void sessionDestroyed(HttpSessionEvent se) {
        System.out.println("Session destroyed: " + se.getSession().getId());
    }
}
```

## 6. 最佳实践

1. 使用MVC架构分离关注点
2. 正确处理字符编码
3. 实现适当的错误处理
4. 使用连接池管理数据库连接
5. 注意安全性考虑

## 总结

本文介绍了Java Web开发中Servlet和JSP的核心概念和实践应用，包括：

1. Servlet的基本概念和生命周期
2. 请求和响应处理
3. JSP的语法和使用
4. MVC架构的实现
5. 过滤器和监听器的应用

通过掌握这些知识，我们可以开发出结构良好的Java Web应用程序。

## 参考资源

1. Java Servlet规范
2. JSP 2.3规范
3. Java Web开发实战
4. Servlet和JSP最佳实践