# Day 32: Java JSON处理 - Jackson和Gson

## 引言

JSON（JavaScript Object Notation）是一种轻量级的数据交换格式，在现代Web应用开发中被广泛使用。Java提供了多种处理JSON的库，其中最受欢迎的是Jackson和Gson。本文将详细介绍这两个库的使用方法和最佳实践。

## 1. Jackson基础

### 1.1 Jackson特点

Jackson是一个功能强大的JSON处理库，具有以下特点：

- 高性能
- 灵活的API
- 支持多种数据格式
- 丰富的注解支持
- 良好的扩展性

### 1.2 Jackson基本使用

```java
import com.fasterxml.jackson.databind.ObjectMapper;

public class JacksonExample {
    public static void main(String[] args) {
        ObjectMapper mapper = new ObjectMapper();
        
        try {
            // Java对象转JSON
            User user = new User("张三", 25);
            String json = mapper.writeValueAsString(user);
            System.out.println(json);
            
            // JSON转Java对象
            String jsonStr = "{\"name\":\"李四\",\"age\":30}";
            User parsedUser = mapper.readValue(jsonStr, User.class);
            System.out.println(parsedUser.getName());
            
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}

class User {
    private String name;
    private int age;
    
    // 构造函数、getter和setter
}
```

### 1.3 Jackson注解

```java
import com.fasterxml.jackson.annotation.*;

@JsonIgnoreProperties(ignoreUnknown = true)
public class Product {
    @JsonProperty("product_name")
    private String name;
    
    @JsonFormat(pattern = "yyyy-MM-dd")
    private Date createTime;
    
    @JsonIgnore
    private String sensitiveInfo;
    
    // getter和setter
}
```

## 2. Gson基础

### 2.1 Gson特点

Gson是Google开发的JSON库，具有以下特点：

- 使用简单
- 性能优秀
- 无需注解
- 支持泛型

### 2.2 Gson基本使用

```java
import com.google.gson.Gson;

public class GsonExample {
    public static void main(String[] args) {
        Gson gson = new Gson();
        
        // Java对象转JSON
        User user = new User("张三", 25);
        String json = gson.toJson(user);
        System.out.println(json);
        
        // JSON转Java对象
        String jsonStr = "{\"name\":\"李四\",\"age\":30}";
        User parsedUser = gson.fromJson(jsonStr, User.class);
        System.out.println(parsedUser.getName());
    }
}
```

### 2.3 Gson高级特性

```java
import com.google.gson.*;

public class GsonAdvanced {
    public static void main(String[] args) {
        // 自定义Gson配置
        Gson gson = new GsonBuilder()
            .setPrettyPrinting()
            .setDateFormat("yyyy-MM-dd")
            .serializeNulls()
            .create();
        
        // 处理泛型
        Type listType = new TypeToken<List<User>>(){}.getType();
        List<User> users = gson.fromJson(jsonArray, listType);
        
        // 自定义序列化
        GsonBuilder builder = new GsonBuilder();
        builder.registerTypeAdapter(Date.class, new DateSerializer());
        Gson customGson = builder.create();
    }
}
```

## 3. 实践案例

### 3.1 RESTful API响应处理

```java
public class ApiResponse<T> {
    private int code;
    private String message;
    private T data;
    
    // 使用Jackson处理
    public static <T> ApiResponse<T> fromJson(String json, Class<T> clazz) {
        ObjectMapper mapper = new ObjectMapper();
        JavaType type = mapper.getTypeFactory()
            .constructParametricType(ApiResponse.class, clazz);
        try {
            return mapper.readValue(json, type);
        } catch (Exception e) {
            throw new RuntimeException("Parse JSON failed", e);
        }
    }
    
    // 使用Gson处理
    public static <T> ApiResponse<T> fromJsonWithGson(String json, Class<T> clazz) {
        Type type = TypeToken.getParameterized(ApiResponse.class, clazz).getType();
        return new Gson().fromJson(json, type);
    }
}
```

### 3.2 JSON配置文件处理

```java
public class ConfigManager {
    private static final String CONFIG_FILE = "config.json";
    private final ObjectMapper mapper;
    private Config config;
    
    public ConfigManager() {
        mapper = new ObjectMapper();
        loadConfig();
    }
    
    private void loadConfig() {
        try {
            config = mapper.readValue(new File(CONFIG_FILE), Config.class);
        } catch (Exception e) {
            throw new RuntimeException("Load config failed", e);
        }
    }
    
    public void saveConfig() {
        try {
            mapper.writerWithDefaultPrettyPrinter()
                  .writeValue(new File(CONFIG_FILE), config);
        } catch (Exception e) {
            throw new RuntimeException("Save config failed", e);
        }
    }
}
```

## 4. 性能优化

1. 对象复用
```java
public class JsonUtil {
    private static final ObjectMapper mapper = new ObjectMapper();
    private static final Gson gson = new Gson();
    
    public static String toJson(Object obj) {
        try {
            return mapper.writeValueAsString(obj);
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }
}
```

2. 批量处理
```java
public class BatchProcessor {
    public void processBatch(List<String> jsonList, Class<?> clazz) {
        ObjectMapper mapper = new ObjectMapper();
        MappingIterator<?> iterator = mapper.readerFor(clazz)
            .readValues(String.join("\n", jsonList));
    }
}
```

## 5. 最佳实践

1. 选择合适的库
   - 简单场景：使用Gson
   - 复杂场景：使用Jackson
   - 性能要求高：根据具体场景测试

2. 异常处理
   - 使用try-catch包装JSON操作
   - 提供合适的错误信息
   - 考虑数据验证

3. 性能考虑
   - 重用ObjectMapper/Gson实例
   - 适当使用缓存
   - 考虑批量处理

4. 安全性
   - 验证输入数据
   - 处理敏感信息
   - 防止JSON注入

## 总结

本文介绍了Java中两个主要的JSON处理库：Jackson和Gson，包括：

1. 基本用法和特性
2. 高级特性和注解
3. 实践案例
4. 性能优化
5. 最佳实践建议

选择合适的JSON处理库并正确使用它们的特性，可以大大提高开发效率和应用性能。

## 参考资源

1. Jackson官方文档：https://github.com/FasterXML/jackson
2. Gson官方文档：https://github.com/google/gson
3. JSON规范：https://www.json.org/
4. Java JSON处理最佳实践指南