# Day 85: Java测试自动化

## 1. 引言

测试自动化是现代软件开发不可或缺的一部分，它能够提高代码质量、减少回归缺陷并加速开发周期。本文将介绍Java测试自动化的核心概念、工具和最佳实践，帮助开发者构建更可靠的软件系统。

## 2. 测试自动化基础

### 2.1 测试金字塔

测试金字塔是一种测试策略模型，从底到顶分为：
- 单元测试（底层）：数量最多，执行最快
- 集成测试（中层）：验证组件间交互
- 端到端测试（顶层）：模拟真实用户场景

### 2.2 测试驱动开发(TDD)

TDD的基本流程：
1. 编写失败的测试
2. 编写最小代码使测试通过
3. 重构代码优化设计

## 3. Java单元测试

### 3.1 JUnit 5基础

```java
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.AfterEach;
import static org.junit.jupiter.api.Assertions.*;

public class CalculatorTest {
    private Calculator calculator;
    
    @BeforeEach
    void setUp() {
        calculator = new Calculator();
    }
    
    @Test
    void testAddition() {
        assertEquals(5, calculator.add(2, 3), "2 + 3 should equal 5");
    }
    
    @Test
    void testDivision() {
        assertEquals(2, calculator.divide(6, 3), "6 / 3 should equal 2");
    }
    
    @Test
    void testDivisionByZero() {
        assertThrows(ArithmeticException.class, () -> calculator.divide(1, 0));
    }
    
    @AfterEach
    void tearDown() {
        calculator = null;
    }
}
```

### 3.2 参数化测试

```java
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.CsvSource;
import org.junit.jupiter.params.provider.ValueSource;

public class StringUtilsTest {
    
    @ParameterizedTest
    @ValueSource(strings = {"racecar", "radar", "level", "refer"})
    void testIsPalindrome(String word) {
        assertTrue(StringUtils.isPalindrome(word));
    }
    
    @ParameterizedTest
    @CsvSource({
        "apple, 5",
        "banana, 6",
        "orange, 6",
        "strawberry, 10"
    })
    void testStringLength(String input, int expectedLength) {
        assertEquals(expectedLength, input.length());
    }
}
```

## 4. 模拟对象

### 4.1 使用Mockito

```java
import org.junit.jupiter.api.Test;
import org.mockito.Mockito;
import static org.mockito.Mockito.*;

public class UserServiceTest {
    
    @Test
    void testCreateUser() {
        // 创建模拟对象
        UserRepository mockRepository = mock(UserRepository.class);
        EmailService mockEmailService = mock(EmailService.class);
        
        // 设置模拟行为
        when(mockRepository.save(any(User.class))).thenReturn(true);
        
        // 创建被测试对象并注入模拟依赖
        UserService userService = new UserService(mockRepository, mockEmailService);
        
        // 执行测试
        boolean result = userService.createUser("test@example.com", "password");
        
        // 验证结果
        assertTrue(result);
        
        // 验证交互
        verify(mockRepository).save(any(User.class));
        verify(mockEmailService).sendWelcomeEmail("test@example.com");
    }
}
```

### 4.2 模拟静态方法

```java
import org.junit.jupiter.api.Test;
import org.mockito.MockedStatic;
import static org.mockito.Mockito.*;

public class UtilityTest {
    
    @Test
    void testStaticMethod() {
        try (MockedStatic<DateUtils> mockedStatic = mockStatic(DateUtils.class)) {
            // 设置静态方法的模拟行为
            mockedStatic.when(DateUtils::getCurrentDate)
                .thenReturn("2023-01-01");
            
            // 使用依赖静态方法的代码
            String result = ReportGenerator.generateDailyReport();
            
            // 验证结果
            assertEquals("Report for 2023-01-01", result);
        }
    }
}
```

## 5. 集成测试

### 5.1 Spring Boot测试

```java
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.test.web.servlet.MockMvc;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@SpringBootTest
@AutoConfigureMockMvc
public class UserControllerIntegrationTest {
    
    @Autowired
    private MockMvc mockMvc;
    
    @Test
    void testGetUserById() throws Exception {
        mockMvc.perform(get("/api/users/1"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.id").value(1))
            .andExpect(jsonPath("$.name").value("John Doe"));
    }
    
    @Test
    void testCreateUser() throws Exception {
        mockMvc.perform(post("/api/users")
            .contentType("application/json")
            .content("{\"name\":\"Jane Smith\",\"email\":\"jane@example.com\"}"))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.id").exists());
    }
}
```

### 5.2 数据库测试

```java
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.jdbc.Sql;

@SpringBootTest
public class UserRepositoryTest {
    
    @Autowired
    private UserRepository userRepository;
    
    @Test
    @Sql("/test-data.sql") // 执行SQL脚本初始化测试数据
    void testFindByEmail() {
        User user = userRepository.findByEmail("test@example.com");
        
        assertNotNull(user);
        assertEquals("Test User", user.getName());
    }
    
    @Test
    void testSaveUser() {
        User newUser = new User();
        newUser.setName("New User");
        newUser.setEmail("new@example.com");
        
        User savedUser = userRepository.save(newUser);
        
        assertNotNull(savedUser.getId());
        assertEquals("New User", savedUser.getName());
    }
}
```

## 6. 端到端测试

### 6.1 使用Selenium WebDriver

```java
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.openqa.selenium.By;
import org.openqa.selenium.WebDriver;
import org.openqa.selenium.WebElement;
import org.openqa.selenium.chrome.ChromeDriver;

import static org.junit.jupiter.api.Assertions.*;

public class LoginPageTest {
    
    private WebDriver driver;
    
    @BeforeEach
    void setUp() {
        System.setProperty("webdriver.chrome.driver", "/path/to/chromedriver");
        driver = new ChromeDriver();
    }
    
    @Test
    void testSuccessfulLogin() {
        // 打开登录页面
        driver.get("https://example.com/login");
        
        // 输入用户名和密码
        WebElement usernameField = driver.findElement(By.id("username"));
        WebElement passwordField = driver.findElement(By.id("password"));
        WebElement loginButton = driver.findElement(By.id("login-button"));
        
        usernameField.sendKeys("testuser");
        passwordField.sendKeys("password");
        loginButton.click();
        
        // 验证登录成功
        WebElement welcomeMessage = driver.findElement(By.id("welcome-message"));
        assertTrue(welcomeMessage.isDisplayed());
        assertEquals("Welcome, Test User!", welcomeMessage.getText());
    }
    
    @AfterEach
    void tearDown() {
        if (driver != null) {
            driver.quit();
        }
    }
}
```

### 6.2 使用Cucumber进行BDD测试

**Feature文件 (login.feature)**:
```gherkin
Feature: User Login

  Scenario: Successful login with valid credentials
    Given I am on the login page
    When I enter username "testuser"
    And I enter password "password"
    And I click the login button
    Then I should be redirected to the dashboard
    And I should see a welcome message "Welcome, Test User!"
```

**步骤定义**:
```java
import io.cucumber.java.After;
import io.cucumber.java.Before;
import io.cucumber.java.en.Given;
import io.cucumber.java.en.When;
import io.cucumber.java.en.Then;
import io.cucumber.java.en.And;
import org.openqa.selenium.By;
import org.openqa.selenium.WebDriver;
import org.openqa.selenium.chrome.ChromeDriver;

import static org.junit.jupiter.api.Assertions.*;

public class LoginStepDefinitions {
    
    private WebDriver driver;
    
    @Before
    public void setUp() {
        System.setProperty("webdriver.chrome.driver", "/path/to/chromedriver");
        driver = new ChromeDriver();
    }
    
    @Given("I am on the login page")
    public void iAmOnTheLoginPage() {
        driver.get("https://example.com/login");
    }
    
    @When("I enter username {string}")
    public void iEnterUsername(String username) {
        driver.findElement(By.id("username")).sendKeys(username);
    }
    
    @And("I enter password {string}")
    public void iEnterPassword(String password) {
        driver.findElement(By.id("password")).sendKeys(password);
    }
    
    @And("I click the login button")
    public void iClickTheLoginButton() {
        driver.findElement(By.id("login-button")).click();
    }
    
    @Then("I should be redirected to the dashboard")
    public void iShouldBeRedirectedToDashboard() {
        assertEquals("Dashboard - Example App", driver.getTitle());
    }
    
    @And("I should see a welcome message {string}")
    public void iShouldSeeWelcomeMessage(String message) {
        String actualMessage = driver.findElement(By.id("welcome-message")).getText();
        assertEquals(message, actualMessage);
    }
    
    @After
    public void tearDown() {
        if (driver != null) {
            driver.quit();
        }
    }
}
```

## 7. 测试覆盖率

### 7.1 使用JaCoCo

**Maven配置**:
```xml
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <version>0.8.8</version>
    <executions>
        <execution>
            <goals>
                <goal>prepare-agent</goal>
            </goals>
        </execution>
        <execution>
            <id>report</id>
            <phase>test</phase>
            <goals>
                <goal>report</goal>
            </goals>
        </execution>
        <execution>
            <id>check</id>
            <goals>
                <goal>check</goal>
            </goals>
            <configuration>
                <rules>
                    <rule>
                        <element>PACKAGE</element>
                        <limits>
                            <limit>
                                <counter>LINE</counter>
                                <value>COVEREDRATIO</value>
                                <minimum>0.80</minimum>
                            </limit>
                        </limits>
                    </rule>
                </rules>
            </configuration>
        </execution>
    </executions>
</plugin>
```

## 8. 持续集成与测试

### 8.1 Jenkins配置

**Jenkinsfile**:
```groovy
pipeline {
    agent any
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Build') {
            steps {
                sh 'mvn clean compile'
            }
        }
        
        stage('Test') {
            steps {
                sh 'mvn test'
            }
            post {
                always {
                    junit '**/target/surefire-reports/*.xml'
                }
            }
        }
        
        stage('Code Coverage') {
            steps {
                sh 'mvn jacoco:report'
            }
            post {
                success {
                    publishHTML(target: [
                        allowMissing: false,
                        alwaysLinkToLastBuild: true,
                        keepAll: true,
                        reportDir: 'target/site/jacoco',
                        reportFiles: 'index.html',
                        reportName: 'JaCoCo Coverage Report'
                    ])
                }
            }
        }
        
        stage('Package') {
            steps {
                sh 'mvn package -DskipTests'
            }
        }
    }
}