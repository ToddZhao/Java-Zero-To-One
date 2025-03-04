# Day 31: Java XML处理 - JAXB和XPath

## 引言

JAXB（Java Architecture for XML Binding）和XPath是Java处理XML的两个重要工具。JAXB提供了Java对象和XML文档之间的映射功能，而XPath则提供了在XML文档中查询和导航的能力。本文将详细介绍这两种技术的使用方法。

## 1. JAXB基础

### 1.1 什么是JAXB

JAXB允许Java开发者将Java类映射到XML表示，主要功能包括：

- 将Java对象转换为XML（编组/Marshal）
- 将XML转换为Java对象（解组/Unmarshal）
- 通过注解定义映射关系
- 支持复杂的数据结构

### 1.2 JAXB注解

```java
import javax.xml.bind.annotation.*;

@XmlRootElement(name = "book")
@XmlAccessorType(XmlAccessType.FIELD)
public class Book {
    @XmlElement(name = "title")
    private String title;
    
    @XmlElement(name = "author")
    private String author;
    
    @XmlAttribute(name = "id")
    private String id;
    
    // getters and setters
}
```

### 1.3 使用JAXB

```java
import javax.xml.bind.*;

public class JAXBExample {
    public static void main(String[] args) {
        try {
            // 创建JAXBContext
            JAXBContext context = JAXBContext.newInstance(Book.class);
            
            // 创建Marshaller
            Marshaller marshaller = context.createMarshaller();
            marshaller.setProperty(Marshaller.JAXB_FORMATTED_OUTPUT, true);
            
            // 创建对象
            Book book = new Book();
            book.setId("1");
            book.setTitle("Java编程思想");
            book.setAuthor("Bruce Eckel");
            
            // 将对象转换为XML
            marshaller.marshal(book, System.out);
            
            // 创建Unmarshaller
            Unmarshaller unmarshaller = context.createUnmarshaller();
            
            // 将XML转换为对象
            Book unmarshalledBook = (Book) unmarshaller.unmarshal(
                new File("book.xml"));
            
        } catch (JAXBException e) {
            e.printStackTrace();
        }
    }
}
```

## 2. XPath技术

### 2.1 XPath简介

XPath是一种在XML文档中查找信息的语言，可以用来：

- 选择XML元素和属性
- 导航XML文档
- 计算值和比较节点

### 2.2 使用XPath

```java
import javax.xml.xpath.*;
import org.w3c.dom.*;

public class XPathExample {
    public static void main(String[] args) {
        try {
            // 创建XPath对象
            XPathFactory xpathFactory = XPathFactory.newInstance();
            XPath xpath = xpathFactory.newXPath();
            
            // 加载XML文档
            DocumentBuilderFactory factory = DocumentBuilderFactory.newInstance();
            DocumentBuilder builder = factory.newDocumentBuilder();
            Document doc = builder.parse("books.xml");
            
            // 使用XPath查询
            // 查找所有book元素
            NodeList books = (NodeList) xpath.evaluate(
                "//book", doc, XPathConstants.NODESET);
            
            // 查找特定属性的book
            String title = xpath.evaluate(
                "//book[@id='1']/title/text()", doc);
            
            // 查找包含特定文本的元素
            Node book = (Node) xpath.evaluate(
                "//book[contains(title, 'Java')]",
                doc, XPathConstants.NODE);
            
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

### 2.3 常用XPath表达式

1. 基本表达式
```
/ - 从根节点选取
// - 从匹配选择的当前节点选择文档中的节点
. - 选取当前节点
.. - 选取当前节点的父节点
@ - 选取属性
```

2. 谓语表达式
```
//book[1] - 选择第一个book元素
//book[@id='1'] - 选择id属性为1的book元素
//book[price>30] - 选择price元素值大于30的book元素
```

## 3. 实践案例

### 3.1 配置文件处理

```java
@XmlRootElement(name = "configuration")
@XmlAccessorType(XmlAccessType.FIELD)
public class Configuration {
    @XmlElementWrapper(name = "settings")
    @XmlElement(name = "setting")
    private List<Setting> settings;
    
    public static class Setting {
        @XmlAttribute
        private String name;
        
        @XmlValue
        private String value;
        
        // getters and setters
    }
    
    public String getValue(String name) {
        return settings.stream()
            .filter(s -> s.name.equals(name))
            .map(s -> s.value)
            .findFirst()
            .orElse(null);
    }
}
```

### 3.2 XML文档查询工具

```java
public class XMLQueryTool {
    private final XPath xpath;
    private final Document document;
    
    public XMLQueryTool(String xmlFile) throws Exception {
        // 初始化XPath
        XPathFactory xpathFactory = XPathFactory.newInstance();
        this.xpath = xpathFactory.newXPath();
        
        // 加载XML文档
        DocumentBuilderFactory factory = DocumentBuilderFactory.newInstance();
        DocumentBuilder builder = factory.newDocumentBuilder();
        this.document = builder.parse(new File(xmlFile));
    }
    
    public String getValue(String xpathExpression) throws XPathExpressionException {
        return xpath.evaluate(xpathExpression, document);
    }
    
    public NodeList getNodes(String xpathExpression) throws XPathExpressionException {
        return (NodeList) xpath.evaluate(xpathExpression, 
            document, XPathConstants.NODESET);
    }
}
```

## 4. 最佳实践

1. JAXB使用建议
   - 使用合适的注解
   - 处理空值和默认值
   - 实现验证逻辑
   - 使用适配器处理特殊类型

2. XPath使用建议
   - 缓存XPath对象
   - 使用命名空间
   - 优化查询表达式
   - 处理大型文档时注意性能

3. 异常处理
   - 处理解析错误
   - 处理验证错误
   - 处理转换错误

4. 性能优化
   - 使用缓存
   - 避免重复解析
   - 使用流式处理

## 总结

本文介绍了Java中处理XML的两个重要工具：JAXB和XPath，包括：

1. JAXB的基本概念和使用方法
2. XPath的查询语法和应用
3. 实践案例
4. 最佳实践建议

通过合理使用这些工具，我们可以更高效地处理XML数据，提高应用程序的可维护性和性能。

## 参考资源

1. JAXB官方文档：https://javaee.github.io/jaxb-v2/
2. XPath规范：https://www.w3.org/TR/xpath/
3. Java XML处理教程：https://docs.oracle.com/javase/tutorial/jaxp/
4. JAXB最佳实践指南