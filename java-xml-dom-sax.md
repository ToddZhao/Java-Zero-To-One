# Day 30: Java XML处理 - DOM和SAX

## 引言

XML（可扩展标记语言）是一种用于存储和传输数据的通用格式。在Java中，我们有多种方式处理XML文档，其中最常用的是DOM和SAX解析器。本文将详细介绍这两种解析方式的使用方法和最佳实践。

## 1. DOM解析

### 1.1 DOM解析原理

DOM（文档对象模型）将XML文档解析成树形结构，可以：

- 读取XML元素
- 修改XML元素
- 删除XML元素
- 创建新的XML元素

### 1.2 使用DOM解析XML

```java
import org.w3c.dom.*;
import javax.xml.parsers.*;
import java.io.File;

public class DOMParser {
    public static void main(String[] args) {
        try {
            // 创建DOM解析器工厂
            DocumentBuilderFactory factory = DocumentBuilderFactory.newInstance();
            DocumentBuilder builder = factory.newDocumentBuilder();

            // 解析XML文件
            Document document = builder.parse(new File("books.xml"));

            // 获取根元素
            Element root = document.getDocumentElement();

            // 获取所有book元素
            NodeList bookList = root.getElementsByTagName("book");

            // 遍历book元素
            for (int i = 0; i < bookList.getLength(); i++) {
                Element book = (Element) bookList.item(i);
                String title = book.getElementsByTagName("title").item(0).getTextContent();
                String author = book.getElementsByTagName("author").item(0).getTextContent();
                System.out.println("Book: " + title + " by " + author);
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

### 1.3 创建XML文档

```java
public class DOMCreator {
    public static void createXML() {
        try {
            DocumentBuilderFactory factory = DocumentBuilderFactory.newInstance();
            DocumentBuilder builder = factory.newDocumentBuilder();
            Document document = builder.newDocument();

            // 创建根元素
            Element root = document.createElement("books");
            document.appendChild(root);

            // 创建book元素
            Element book = document.createElement("book");
            root.appendChild(book);

            // 添加子元素
            Element title = document.createElement("title");
            title.setTextContent("Java编程思想");
            book.appendChild(title);

            // 保存XML文件
            TransformerFactory transformerFactory = TransformerFactory.newInstance();
            Transformer transformer = transformerFactory.newTransformer();
            DOMSource source = new DOMSource(document);
            StreamResult result = new StreamResult(new File("newbooks.xml"));
            transformer.transform(source, result);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

## 2. SAX解析

### 2.1 SAX解析原理

SAX（Simple API for XML）是一个基于事件的解析器，具有以下特点：

- 顺序读取XML文件
- 不需要将整个文档加载到内存
- 适合处理大型XML文件
- 只能读取XML，不能修改

### 2.2 使用SAX解析XML

```java
import org.xml.sax.*;
import org.xml.sax.helpers.*;
import javax.xml.parsers.*;

public class SAXParser extends DefaultHandler {
    private StringBuilder currentValue = new StringBuilder();

    @Override
    public void startElement(String uri, String localName, String qName, Attributes attributes) {
        // 清空当前值
        currentValue.setLength(0);
        
        // 处理元素开始标签
        System.out.println("Start Element: " + qName);
    }

    @Override
    public void characters(char[] ch, int start, int length) {
        // 累加当前值
        currentValue.append(ch, start, length);
    }

    @Override
    public void endElement(String uri, String localName, String qName) {
        // 处理元素结束标签
        System.out.println("End Element: " + qName);
        System.out.println("Element Value: " + currentValue.toString().trim());
    }

    public static void main(String[] args) {
        try {
            SAXParserFactory factory = SAXParserFactory.newInstance();
            javax.xml.parsers.SAXParser saxParser = factory.newSAXParser();
            SAXParser handler = new SAXParser();
            saxParser.parse("books.xml", handler);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

## 3. DOM vs SAX比较

### 3.1 DOM优势

1. 可以随机访问文档的任何部分
2. 可以修改文档结构
3. 编程模型简单直观

### 3.2 SAX优势

1. 内存占用小
2. 处理速度快
3. 适合处理大型文件

## 4. 实践案例

### 4.1 配置文件解析器

```java
public class ConfigParser {
    private Map<String, String> config = new HashMap<>();

    public void parseConfig(String filename) {
        try {
            DocumentBuilderFactory factory = DocumentBuilderFactory.newInstance();
            DocumentBuilder builder = factory.newDocumentBuilder();
            Document document = builder.parse(new File(filename));

            NodeList items = document.getElementsByTagName("item");
            for (int i = 0; i < items.getLength(); i++) {
                Element item = (Element) items.item(i);
                String key = item.getAttribute("key");
                String value = item.getTextContent();
                config.put(key, value);
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    public String getConfig(String key) {
        return config.get(key);
    }
}
```

## 5. 最佳实践

1. 选择合适的解析器
   - 小型XML文件：使用DOM
   - 大型XML文件：使用SAX
   - 需要修改XML：使用DOM

2. 异常处理
   - 处理格式错误
   - 处理文件访问异常
   - 处理空值情况

3. 性能优化
   - 使用缓冲读取
   - 避免重复解析
   - 及时释放资源

4. 安全考虑
   - 防止XXE攻击
   - 验证XML格式
   - 限制文件大小

## 总结

本文介绍了Java中XML处理的两种主要方式：DOM和SAX，包括：

1. DOM解析的原理和使用
2. SAX解析的原理和使用
3. 两种方式的比较
4. 实践案例
5. 最佳实践建议

选择合适的XML解析方式对于提高应用程序的性能和可维护性非常重要。

## 参考资源

1. Java XML教程：https://docs.oracle.com/javase/tutorial/jaxp/
2. DOM API文档：https://docs.oracle.com/javase/8/docs/api/org/w3c/dom/package-summary.html
3. SAX API文档：https://docs.oracle.com/javase/8/docs/api/org/xml/sax/package-summary.html
4. XML处理最佳实践指南