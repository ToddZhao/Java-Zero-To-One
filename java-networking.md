# Day 5: Java网络编程

## 引言

Java网络编程是现代应用开发中不可或缺的一部分。它使应用程序能够通过网络进行通信，实现分布式系统和互联网应用。本文将详细介绍Java网络编程的核心概念和实践应用。

## 1. 网络编程基础

### 1.1 网络基本概念

- IP地址：网络设备的唯一标识
- 端口：应用程序的通信端点
- 协议：通信规则（如TCP、UDP）

### 1.2 Socket编程

Socket是网络编程的基础，提供了主机间通信的端点：

```java
// 创建服务器Socket
ServerSocket serverSocket = new ServerSocket(8080);

// 创建客户端Socket
Socket clientSocket = new Socket("localhost", 8080);
```

## 2. TCP通信

### 2.1 TCP服务器

```java
public class TCPServer {
    public static void main(String[] args) {
        try (ServerSocket serverSocket = new ServerSocket(8080)) {
            System.out.println("服务器启动，等待连接...");
            
            while (true) {
                Socket clientSocket = serverSocket.accept();
                new Thread(() -> handleClient(clientSocket)).start();
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
    
    private static void handleClient(Socket clientSocket) {
        try (BufferedReader reader = new BufferedReader(
                new InputStreamReader(clientSocket.getInputStream()));
             PrintWriter writer = new PrintWriter(
                clientSocket.getOutputStream(), true)) {
            
            String line;
            while ((line = reader.readLine()) != null) {
                System.out.println("收到消息：" + line);
                writer.println("服务器回复：" + line);
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

### 2.2 TCP客户端

```java
public class TCPClient {
    public static void main(String[] args) {
        try (Socket socket = new Socket("localhost", 8080);
             BufferedReader reader = new BufferedReader(
                new InputStreamReader(socket.getInputStream()));
             PrintWriter writer = new PrintWriter(
                socket.getOutputStream(), true);
             BufferedReader consoleReader = new BufferedReader(
                new InputStreamReader(System.in))) {
            
            String userInput;
            while ((userInput = consoleReader.readLine()) != null) {
                writer.println(userInput);
                System.out.println(reader.readLine());
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

## 3. UDP通信

### 3.1 UDP服务器

```java
public class UDPServer {
    public static void main(String[] args) {
        try (DatagramSocket socket = new DatagramSocket(8080)) {
            byte[] buffer = new byte[1024];
            
            while (true) {
                DatagramPacket packet = new DatagramPacket(buffer, buffer.length);
                socket.receive(packet);
                
                String received = new String(
                    packet.getData(), 0, packet.getLength());
                System.out.println("收到消息：" + received);
                
                // 发送回复
                String response = "服务器回复：" + received;
                byte[] responseData = response.getBytes();
                DatagramPacket responsePacket = new DatagramPacket(
                    responseData,
                    responseData.length,
                    packet.getAddress(),
                    packet.getPort());
                socket.send(responsePacket);
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

### 3.2 UDP客户端

```java
public class UDPClient {
    public static void main(String[] args) {
        try (DatagramSocket socket = new DatagramSocket()) {
            InetAddress address = InetAddress.getByName("localhost");
            BufferedReader reader = new BufferedReader(
                new InputStreamReader(System.in));
            
            String userInput;
            while ((userInput = reader.readLine()) != null) {
                byte[] sendData = userInput.getBytes();
                DatagramPacket sendPacket = new DatagramPacket(
                    sendData,
                    sendData.length,
                    address,
                    8080);
                socket.send(sendPacket);
                
                // 接收响应
                byte[] receiveData = new byte[1024];
                DatagramPacket receivePacket = new DatagramPacket(
                    receiveData,
                    receiveData.length);
                socket.receive(receivePacket);
                
                String response = new String(
                    receivePacket.getData(), 
                    0, 
                    receivePacket.getLength());
                System.out.println(response);
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

## 4. URL和URLConnection

### 4.1 URL操作

```java
public class URLExample {
    public static String getContent(String urlString) {
        try {
            URL url = new URL(urlString);
            try (BufferedReader reader = new BufferedReader(
                    new InputStreamReader(url.openStream()))) {
                
                StringBuilder content = new StringBuilder();
                String line;
                while ((line = reader.readLine()) != null) {
                    content.append(line).append("\n");
                }
                return content.toString();
            }
        } catch (IOException e) {
            e.printStackTrace();
            return null;
        }
    }
}
```

### 4.2 HTTPURLConnection

```java
public class HTTPExample {
    public static String sendGet(String urlString) {
        try {
            URL url = new URL(urlString);
            HttpURLConnection conn = (HttpURLConnection) url.openConnection();
            conn.setRequestMethod("GET");
            
            try (BufferedReader reader = new BufferedReader(
                    new InputStreamReader(conn.getInputStream()))) {
                
                StringBuilder response = new StringBuilder();
                String line;
                while ((line = reader.readLine()) != null) {
                    response.append(line);
                }
                return response.toString();
            }
        } catch (IOException e) {
            e.printStackTrace();
            return null;
        }
    }
}
```

## 5. 网络编程最佳实践

1. 正确处理网络异常
2. 使用线程池处理并发连接
3. 设置合适的超时时间
4. 及时关闭网络资源
5. 使用缓冲流提高性能

## 总结

本文介绍了Java网络编程的核心概念和实践应用，包括：

1. 网络编程基础知识
2. TCP通信的实现
3. UDP通信的实现
4. URL和HTTP连接的使用
5. 网络编程最佳实践

通过掌握这些知识，我们可以开发出高效、可靠的网络应用程序。

## 参考资源

1. Java官方文档：https://docs.oracle.com/javase/tutorial/networking/
2. Java网络编程指南
3. TCP/IP协议详解
4. Java网络编程最佳实践