# Day 84: Java物联网开发

## 1. 引言

物联网（Internet of Things，IoT）是指通过互联网将各种设备连接起来，实现信息交换和通信的网络。Java作为一种跨平台的编程语言，在物联网开发中扮演着重要角色。本文将介绍如何使用Java进行物联网应用开发，并探讨相关技术和最佳实践。

## 2. Java在物联网中的应用

### 2.1 为什么选择Java

在物联网开发中，Java具有以下优势：
- 跨平台性：一次编写，到处运行
- 丰富的库和框架支持
- 强大的安全特性
- 成熟的网络编程能力
- 良好的并发处理能力

### 2.2 Java物联网技术栈

- Java ME (Micro Edition)：适用于资源受限设备
- Java SE Embedded：适用于嵌入式系统
- Spring Boot：构建后端服务和API
- Eclipse Paho：MQTT客户端库
- Eclipse Kura：IoT网关框架
- Eclipse Milo：OPC UA协议实现

## 3. 物联网通信协议

### 3.1 MQTT协议

MQTT（Message Queuing Telemetry Transport）是一种轻量级的发布/订阅消息传输协议，特别适合物联网设备间的通信。

#### 使用Eclipse Paho实现MQTT客户端

```java
public class MqttPublisher {
    public static void main(String[] args) {
        String broker = "tcp://mqtt.eclipse.org:1883";
        String clientId = "JavaSample";
        String topic = "sensors/temperature";
        String content = "25.5";
        int qos = 2;
        
        try {
            MqttClient client = new MqttClient(broker, clientId);
            MqttConnectOptions options = new MqttConnectOptions();
            options.setCleanSession(true);
            
            System.out.println("Connecting to broker: " + broker);
            client.connect(options);
            System.out.println("Connected");
            
            System.out.println("Publishing message: " + content);
            MqttMessage message = new MqttMessage(content.getBytes());
            message.setQos(qos);
            client.publish(topic, message);
            System.out.println("Message published");
            
            client.disconnect();
            System.out.println("Disconnected");
        } catch (MqttException me) {
            System.out.println("reason " + me.getReasonCode());
            System.out.println("msg " + me.getMessage());
            System.out.println("loc " + me.getLocalizedMessage());
            System.out.println("cause " + me.getCause());
            System.out.println("excep " + me);
            me.printStackTrace();
        }
    }
}
```

#### MQTT订阅者实现

```java
public class MqttSubscriber implements MqttCallback {
    private final String broker = "tcp://mqtt.eclipse.org:1883";
    private final String clientId = "JavaSubscriber";
    private final String topic = "sensors/temperature";
    private MqttClient client;
    
    public MqttSubscriber() {
        try {
            client = new MqttClient(broker, clientId);
            client.setCallback(this);
            MqttConnectOptions options = new MqttConnectOptions();
            options.setCleanSession(true);
            
            System.out.println("Connecting to broker: " + broker);
            client.connect(options);
            System.out.println("Connected");
            
            client.subscribe(topic);
            System.out.println("Subscribed to topic: " + topic);
        } catch (MqttException me) {
            me.printStackTrace();
        }
    }
    
    @Override
    public void connectionLost(Throwable cause) {
        System.out.println("Connection lost: " + cause.getMessage());
    }
    
    @Override
    public void messageArrived(String topic, MqttMessage message) throws Exception {
        String payload = new String(message.getPayload());
        System.out.println("Received message: " + payload + " from topic: " + topic);
        // 处理接收到的数据
        processTemperatureData(payload);
    }
    
    @Override
    public void deliveryComplete(IMqttDeliveryToken token) {
        // 消息发送完成回调
    }
    
    private void processTemperatureData(String temperature) {
        // 处理温度数据的逻辑
        try {
            double temp = Double.parseDouble(temperature);
            if (temp > 30.0) {
                System.out.println("警告：温度过高！");
                // 触发警报或其他操作
            }
        } catch (NumberFormatException e) {
            System.out.println("无效的温度数据格式");
        }
    }
    
    public static void main(String[] args) {
        new MqttSubscriber();
        // 保持程序运行以接收消息
        try {
            Thread.sleep(Long.MAX_VALUE);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

### 3.2 CoAP协议

CoAP（Constrained Application Protocol）是一种专为资源受限设备设计的协议，类似于HTTP但更轻量级。

```java
public class CoapClient {
    public static void main(String[] args) {
        // 创建CoAP客户端
        org.eclipse.californium.core.CoapClient client = new org.eclipse.californium.core.CoapClient("coap://coap.me:5683/hello");
        
        // 发送GET请求
        CoapResponse response = client.get();
        
        if (response != null) {
            System.out.println("Response Code: " + response.getCode());
            System.out.println("Response Text: " + response.getResponseText());
        } else {
            System.out.println("No response received.");
        }
    }
}
```

## 4. 传感器数据处理

### 4.1 数据采集

```java
public class SensorDataCollector {
    private final SerialPort serialPort;
    private final InputStream inputStream;
    
    public SensorDataCollector(String portName) throws Exception {
        // 初始化串口通信
        serialPort = SerialPort.getCommPort(portName);
        serialPort.setComPortParameters(9600, 8, 1, SerialPort.NO_PARITY);
        serialPort.setComPortTimeouts(SerialPort.TIMEOUT_READ_SEMI_BLOCKING, 1000, 0);
        
        if (!serialPort.openPort()) {
            throw new Exception("无法打开串口: " + portName);
        }
        
        inputStream = serialPort.getInputStream();
    }
    
    public String readData() throws IOException {
        byte[] buffer = new byte[1024];
        int len = inputStream.read(buffer);
        
        if (len > 0) {
            return new String(buffer, 0, len);
        }
        
        return null;
    }
    
    public void close() {
        if (serialPort != null) {
            serialPort.closePort();
        }
    }
}
```

### 4.2 数据处理与分析

```java
public class SensorDataProcessor {
    private final DataSource dataSource;
    
    public SensorDataProcessor(DataSource dataSource) {
        this.dataSource = dataSource;
    }
    
    public void processBatchData() throws SQLException {
        Connection conn = dataSource.getConnection();
        try {
            // 查询最近一小时的数据
            PreparedStatement stmt = conn.prepareStatement(
                "SELECT sensor_id, AVG(value) as avg_value, MAX(value) as max_value, MIN(value) as min_value " +
                "FROM sensor_data " +
                "WHERE timestamp > ? " +
                "GROUP BY sensor_id");
            
            Timestamp oneHourAgo = new Timestamp(System.currentTimeMillis() - 3600 * 1000);
            stmt.setTimestamp(1, oneHourAgo);
            
            ResultSet rs = stmt.executeQuery();
            while (rs.next()) {
                String sensorId = rs.getString("sensor_id");
                double avgValue = rs.getDouble("avg_value");
                double maxValue = rs.getDouble("max_value");
                double minValue = rs.getDouble("min_value");
                
                System.out.println("Sensor: " + sensorId);
                System.out.println("  Average: " + avgValue);
                System.out.println("  Maximum: " + maxValue);
                System.out.println("  Minimum: " + minValue);
                
                // 检测异常值
                if (maxValue > getThreshold(sensorId)) {
                    triggerAlert(sensorId, maxValue);
                }
            }
        } finally {
            conn.close();
        }
    }
    
    private double getThreshold(String sensorId) {
        // 获取传感器阈值的逻辑
        return 100.0; // 示例阈值
    }
    
    private void triggerAlert(String sensorId, double value) {
        // 触发警报的逻辑
        System.out.println("ALERT: Sensor " + sensorId + " reported high value: " + value);
        // 发送通知或执行其他操作
    }
}
```

## 5. 物联网网关开发

### 5.1 使用Eclipse Kura构建网关

```java
public class IoTGateway extends AbstractCloudService {
    private static final Logger logger = LoggerFactory.getLogger(IoTGateway.class);
    private static final String APP_ID = "org.example.iot.gateway";
    
    private CloudClient cloudClient;
    private Map<String, SensorReader> sensors = new HashMap<>();
    
    protected void activate(ComponentContext componentContext) {
        logger.info("激活IoT网关服务");
        // 初始化云连接
        try {
            cloudClient = getCloudService().newCloudClient(APP_ID);
            cloudClient.addCloudClientListener(this);
        } catch (KuraException e) {
            logger.error("云客户端初始化失败", e);
        }
        
        // 初始化传感器
        initSensors();
    }
    
    protected void deactivate(ComponentContext componentContext) {
        logger.info("停用IoT网关服务");
        // 关闭传感器连接
        for (SensorReader sensor : sensors.values()) {
            sensor.stop();
        }
        sensors.clear();
        
        // 释放云客户端
        if (cloudClient != null) {
            cloudClient.release();
        }
    }
    
    private void initSensors() {
        // 初始化温度传感器
        SensorReader tempSensor = new TemperatureSensorReader("temp-001");
        tempSensor.setDataHandler(this::publishSensorData);
        tempSensor.start();
        sensors.put("temp-001", tempSensor);
        
        // 初始化湿度传感器
        SensorReader humiditySensor = new HumiditySensorReader("hum-001");
        humiditySensor.setDataHandler(this::publishSensorData);
        humiditySensor.start();
        sensors.put("hum-001", humiditySensor);
    }
    
    private void publishSensorData(String sensorId, double value) {
        try {
            // 创建KuraPayload
            KuraPayload payload = new KuraPayload();
            payload.addMetric("value", value);
            payload.addMetric("timestamp", System.currentTimeMillis());
            payload.addMetric("sensorId", sensorId);
            
            // 发布到云平台
            cloudClient.publish("sensors/" + sensorId, payload, 0, false);
            logger.info("已发布传感器数据: {} = {}", sensorId, value);
        } catch (KuraException e) {
            logger.error("发布传感器数据失败", e);
        }
    }
    
    @Override
    public void onConnectionLost() {
        logger.warn("云连接丢失");
    }
    
    @Override
    public void onConnectionEstablished() {
        logger.info("云连接已建立");
    }
    
    @Override
    public void onMessageArrived(String topic, byte[] payload, int qos, boolean retained) {
        logger.info("收到消息: {}", topic);
        // 处理下行命令
        if (topic.startsWith("command/")) {
            String[] parts = topic.split("/");
            if (parts.length >= 2) {
                String sensorId = parts[1];
                handleCommand(sensorId, payload);
            }
        }
    }
    
    private void handleCommand(String sensorId, byte[] payload) {
        // 处理下行命令的逻辑