# Day 4: Java IO和文件操作

## 引言

Java IO（输入/输出）是Java编程中最基础且重要的部分之一。它提供了一套完整的机制来处理数据的输入和输出，无论是文件操作、网络通信还是内存数据处理。本文将详细介绍Java IO的核心概念和实践应用。

## 1. Java IO基础

### 1.1 IO流的概念

Java中的IO操作主要是通过流(Stream)来实现的。流是一组有顺序的数据序列，分为输入流和输出流：

- 输入流：从外部读取数据到程序中
- 输出流：从程序中写出数据到外部

### 1.2 IO流的分类

按照处理数据的单位分类：
- 字节流：以字节为单位处理数据
- 字符流：以字符为单位处理数据

## 2. 常用IO流类

### 2.1 字节流

```java
// 文件字节输入流
FileInputStream fis = new FileInputStream("input.txt");

// 文件字节输出流
FileOutputStream fos = new FileOutputStream("output.txt");

// 缓冲字节流
BufferedInputStream bis = new BufferedInputStream(fis);
BufferedOutputStream bos = new BufferedOutputStream(fos);
```

### 2.2 字符流

```java
// 文件字符输入流
FileReader fr = new FileReader("input.txt");

// 文件字符输出流
FileWriter fw = new FileWriter("output.txt");

// 缓冲字符流
BufferedReader br = new BufferedReader(fr);
BufferedWriter bw = new BufferedWriter(fw);
```

## 3. 文件操作

### 3.1 File类

```java
// 创建文件对象
File file = new File("test.txt");

// 创建文件
boolean created = file.createNewFile();

// 创建目录
File dir = new File("testDir");
boolean dirCreated = dir.mkdir();

// 文件属性操作
boolean exists = file.exists();
boolean isFile = file.isFile();
boolean isDirectory = file.isDirectory();
long length = file.length();
```

### 3.2 文件读写示例

```java
public class FileOperationExample {
    public static void writeFile(String content, String fileName) {
        try (BufferedWriter writer = new BufferedWriter(new FileWriter(fileName))) {
            writer.write(content);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    public static String readFile(String fileName) {
        StringBuilder content = new StringBuilder();
        try (BufferedReader reader = new BufferedReader(new FileReader(fileName))) {
            String line;
            while ((line = reader.readLine()) != null) {
                content.append(line).append("\n");
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
        return content.toString();
    }
}
```

## 4. NIO（New IO）

### 4.1 NIO的核心组件

- Buffer（缓冲区）
- Channel（通道）
- Selector（选择器）

### 4.2 NIO示例

```java
public class NIOExample {
    public static void copyFile(String source, String target) {
        try (FileChannel sourceChannel = FileChannel.open(Paths.get(source), StandardOpenOption.READ);
             FileChannel targetChannel = FileChannel.open(Paths.get(target), 
                 StandardOpenOption.CREATE, StandardOpenOption.WRITE)) {
            
            ByteBuffer buffer = ByteBuffer.allocate(1024);
            while (sourceChannel.read(buffer) > 0) {
                buffer.flip();
                targetChannel.write(buffer);
                buffer.clear();
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

## 5. 实践案例

### 5.1 文件复制工具

```java
public class FileCopyUtil {
    public static void copyFile(String sourcePath, String targetPath) {
        try (InputStream in = new FileInputStream(sourcePath);
             OutputStream out = new FileOutputStream(targetPath)) {
            
            byte[] buffer = new byte[1024];
            int length;
            while ((length = in.read(buffer)) > 0) {
                out.write(buffer, 0, length);
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

### 5.2 日志记录器

```java
public class SimpleLogger {
    private static final String LOG_FILE = "application.log";

    public static void log(String message) {
        try (BufferedWriter writer = new BufferedWriter(
                new FileWriter(LOG_FILE, true))) {
            writer.write(String.format("%s: %s\n", 
                LocalDateTime.now(), message));
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

## 6. 最佳实践

1. 始终使用try-with-resources语句处理IO资源
2. 使用缓冲流提高IO性能
3. 根据数据类型选择合适的流
4. 正确处理异常
5. 及时关闭资源

## 总结

本文介绍了Java IO和文件操作的核心概念和实践应用，包括：

1. IO流的基本概念和分类
2. 常用IO流类的使用
3. 文件操作的基本方法
4. NIO的核心组件和使用
5. 实践案例和最佳实践

通过掌握这些知识，我们可以更好地处理程序中的输入输出操作，提高程序的性能和可靠性。

## 参考资源

1. Java官方文档：https://docs.oracle.com/javase/tutorial/essential/io/
2. Java IO最佳实践指南
3. Java NIO编程指南
4. Effective Java中的IO处理建议