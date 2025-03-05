# Day 125: Java云原生高级 - Serverless架构

## 1. Serverless架构概述

Serverless（无服务器）架构是一种云计算执行模型，它允许开发者构建和运行应用程序而无需管理服务器。在Serverless架构中，云提供商负责执行代码所需的基础设施，并根据实际使用量进行计费。这种架构模式特别适合事件驱动型应用和微服务。

## 2. Serverless核心概念

### 2.1 函数即服务（FaaS）

```java
// AWS Lambda函数示例
public class OrderProcessorFunction implements RequestHandler<APIGatewayProxyRequestEvent, APIGatewayProxyResponseEvent> {
    
    private final AmazonDynamoDB dynamoDBClient = AmazonDynamoDBClientBuilder.standard().build();
    private final DynamoDB dynamoDB = new DynamoDB(dynamoDBClient);
    private final Table orderTable = dynamoDB.getTable(System.getenv("ORDER_TABLE_NAME"));
    private final ObjectMapper objectMapper = new ObjectMapper();
    
    @Override
    public APIGatewayProxyResponseEvent handleRequest(APIGatewayProxyRequestEvent input, Context context) {
        LambdaLogger logger = context.getLogger();
        APIGatewayProxyResponseEvent response = new APIGatewayProxyResponseEvent();
        
        try {
            // 解析请求体
            OrderRequest orderRequest = objectMapper.readValue(input.getBody(), OrderRequest.class);
            
            // 生成订单ID
            String orderId = UUID.randomUUID().toString();
            
            // 创建订单项
            Item orderItem = new Item()
                .withPrimaryKey("id", orderId)
                .withString("customerId", orderRequest.getCustomerId())
                .withString("status", "CREATED")
                .withNumber("amount", orderRequest.getAmount())
                .withString("createdAt", Instant.now().toString());
            
            // 保存订单到DynamoDB
            orderTable.putItem(orderItem);
            
            // 构建响应
            OrderResponse orderResponse = new OrderResponse(orderId, "Order created successfully");
            String responseBody = objectMapper.writeValueAsString(orderResponse);
            
            response.setStatusCode(201);
            response.setBody(responseBody);
            response.setHeaders(Collections.singletonMap("Content-Type", "application/json"));
            
            logger.log("Order created: " + orderId);
            
        } catch (Exception e) {
            logger.log("Error processing request: " + e.getMessage());
            
            response.setStatusCode(500);
            response.setBody("{\"error\":\"Internal Server Error\"}");
        }
        
        return response;
    }
}
```

### 2.2 事件触发

```java
// AWS S3事件触发Lambda函数
public class ImageProcessorFunction implements RequestHandler<S3Event, String> {
    
    private final AmazonS3 s3Client = AmazonS3ClientBuilder.standard().build();
    private final AmazonSQS sqsClient = AmazonSQSClientBuilder.standard().build();
    private final String outputQueueUrl = System.getenv("OUTPUT_QUEUE_URL");
    
    @Override
    public String handleRequest(S3Event event, Context context) {
        LambdaLogger logger = context.getLogger();
        
        for (S3EventNotification.S3EventNotificationRecord record : event.getRecords()) {
            String bucket = record.getS3().getBucket().getName();
            String key = record.getS3().getObject().getKey();
            
            logger.log("Processing image: " + key + " from bucket: " + bucket);
            
            try {
                // 下载图片
                S3Object s3Object = s3Client.getObject(bucket, key);
                BufferedImage originalImage = ImageIO.read(s3Object.getObjectContent());
                
                // 处理图片（例如调整大小）
                BufferedImage resizedImage = resizeImage(originalImage, 300, 300);
                
                // 上传处理后的图片
                String resizedKey = "resized/" + key;
                uploadImage(resizedImage, bucket, resizedKey);
                
                // 发送处理完成通知到SQS
                Map<String, String> messageAttributes = new HashMap<>();
                messageAttributes.put("originalKey", key);
                messageAttributes.put("resizedKey", resizedKey);
                
                String messageBody = new ObjectMapper().writeValueAsString(messageAttributes);
                sqsClient.sendMessage(outputQueueUrl, messageBody);
                
                logger.log("Image processed successfully: " + resizedKey);
                
            } catch (Exception e) {
                logger.log("Error processing image: " + e.getMessage());
                return "Error: " + e.getMessage();
            }
        }
        
        return "Image processing completed";
    }
    
    private BufferedImage resizeImage(BufferedImage originalImage, int targetWidth, int targetHeight) {
        BufferedImage resizedImage = new BufferedImage(targetWidth, targetHeight, BufferedImage.TYPE_INT_RGB);
        Graphics2D graphics = resizedImage.createGraphics();
        graphics.drawImage(originalImage, 0, 0, targetWidth, targetHeight, null);
        graphics.dispose();
        return resizedImage;
    }
    
    private void uploadImage(BufferedImage image, String bucket, String key) throws IOException {
        ByteArrayOutputStream outputStream = new ByteArrayOutputStream();
        ImageIO.write(image, "jpg", outputStream);
        byte[] imageBytes = outputStream.toByteArray();
        
        ObjectMetadata metadata = new ObjectMetadata();
        metadata.setContentLength(imageBytes.length);
        metadata.setContentType("image/jpeg");
        
        s3Client.putObject(bucket, key, new ByteArrayInputStream(imageBytes), metadata);
    }
}
```

## 3. Serverless框架与工具

### 3.1 AWS SAM (Serverless Application Model)

```yaml
# template.yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: Order Processing Serverless Application

Globals:
  Function:
    Timeout: 30
    MemorySize: 512
    Runtime: java11
    Handler: com.example.OrderProcessorFunction::handleRequest

Resources:
  OrderTable:
    Type: AWS::DynamoDB::Table
    Properties:
      TableName: orders
      BillingMode: PAY_PER_REQUEST
      AttributeDefinitions:
        - AttributeName: id
          AttributeType: S
      KeySchema:
        - AttributeName: id
          KeyType: HASH

  OrderProcessorFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: ./target/order-processor-1.0.0.jar
      Policies:
        - DynamoDBCrudPolicy:
            TableName: !Ref OrderTable
      Environment:
        Variables:
          ORDER_TABLE_NAME: !Ref OrderTable
      Events:
        ApiEvent:
          Type: Api
          Properties:
            Path: /orders
            Method: post

  ImageProcessorFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: ./target/image-processor-1.0.0.jar
      Handler: com.example.ImageProcessorFunction::handleRequest
      Policies:
        - S3ReadPolicy:
            BucketName: !Ref ImageBucket
        - S3WritePolicy:
            BucketName: !Ref ImageBucket
        - SQSSendMessagePolicy:
            QueueName: !GetAtt ProcessingQueue.QueueName
      Environment:
        Variables:
          OUTPUT_QUEUE_URL: !Ref ProcessingQueue
      Events:
        S3Event:
          Type: S3
          Properties:
            Bucket: !Ref ImageBucket
            Events: s3:ObjectCreated:*
            Filter:
              S3Key:
                Rules:
                  - Name: prefix
                    Value: uploads/

  ImageBucket:
    Type: AWS::S3::Bucket

  ProcessingQueue:
    Type: AWS::SQS::Queue
    Properties:
      VisibilityTimeout: 300

Outputs:
  ApiEndpoint:
    Description: API Gateway endpoint URL
    Value: !Sub https://${ServerlessRestApi}.execute-api.${AWS::Region}.amazonaws.com/Prod/orders
  ImageBucketName:
    Description: S3 Bucket for image uploads
    Value: !Ref ImageBucket
```

### 3.2 Spring Cloud Function

```java
@SpringBootApplication
public class ServerlessApplication {

    public static void main(String[] args) {
        SpringApplication.run(ServerlessApplication.class, args);
    }
    
    @Bean
    public Function<String, String> uppercase() {
        return value -> value.toUpperCase();
    }
    
    @Bean
    public Function<Flux<String>, Flux<String>> reverseFlux() {
        return flux -> flux.map(value -> new StringBuilder(value).reverse().toString());
    }
    
    @Bean
    public Consumer<Message<OrderEvent>> processOrder() {
        return message -> {
            OrderEvent event = message.getPayload();
            System.out.println("Processing order: " + event.getOrderId());
            // 处理订单逻辑
        };
    }
    
    @Bean
    public Supplier<Flux<String>> heartbeat() {
        return () -> Flux.interval(Duration.ofSeconds(1))
                .map(i -> "Heartbeat " + LocalDateTime.now());
    }
}
```

## 4. Serverless数据持久化

### 4.1 DynamoDB集成

```java
@Service
public class OrderRepository {
    
    private final DynamoDBMapper dynamoDBMapper;
    
    public OrderRepository(AmazonDynamoDB dynamoDBClient) {
        this.dynamoDBMapper = new DynamoDBMapper(dynamoDBClient);
    }
    
    public Order save(Order order) {
        dynamoDBMapper.save(order);
        return order;
    }
    
    public Optional<Order> findById(String orderId) {
        Order order = dynamoDBMapper.load(Order.class, orderId);
        return Optional.ofNullable(order);
    }
    
    public List<Order> findByCustomerId(String customerId) {
        Map<String, AttributeValue> expressionAttributeValues = new HashMap<>();
        expressionAttributeValues.put(":customerId", new AttributeValue().withS(customerId));
        
        DynamoDBQueryExpression<Order> queryExpression = new DynamoDBQueryExpression<Order>()
                .withIndexName("customerIdIndex")
                .withConsistentRead(false)
                .withKeyConditionExpression("customerId = :customerId")
                .withExpressionAttributeValues(expressionAttributeValues);
        
        return dynamoDBMapper.query(Order.class, queryExpression);
    }
    
    public void delete(Order order) {
        dynamoDBMapper.delete(order);
    }
}

@DynamoDBTable(tableName = "orders")
public class Order {
    
    @DynamoDBHashKey
    private String id;
    
    @DynamoDBIndexHashKey(globalSecondaryIndexName = "customerIdIndex")
    private String customerId;
    
    private String status;
    private BigDecimal amount;
    
    @DynamoDBTypeConverted(converter = InstantConverter.class)
    private Instant createdAt;
    
    // getters and setters
    
    public static class InstantConverter implements DynamoDBTypeConverter<String, Instant> {
        @Override
        public String convert(Instant instant) {
            return instant.toString();
        }
        
        @Override
        public Instant unconvert(String value) {
            return Instant.parse(value);
        }
    }
}
```

### 4.2 Aurora Serverless集成

```java
@Configuration
public class DatabaseConfig {
    
    @Bean
    public DataSource dataSource() {
        RdsIamAuthTokenGenerator tokenGenerator = RdsIamAuthTokenGenerator.builder()
                .credentials(new DefaultAWSCredentialsProviderChain())
                .region(Regions.getCurrentRegion().getName())
                .build();
        
        String hostname = System.getenv("DB_HOSTNAME");
        String port = System.getenv("DB_PORT");
        String dbname = System.getenv("DB_NAME");
        String username = System.getenv("DB_USERNAME");
        
        String token = tokenGenerator.getAuthToken(
                GetIamAuthTokenRequest.builder()
                        .hostname(hostname)
                        .port(Integer.parseInt(port))
                        .userName(username)
                        .build());
        
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl(String.format("jdbc:mysql://%s:%s/%s", hostname, port, dbname));
        config.setUsername(username);
        config.setPassword(token);
        config.addDataSourceProperty("useSSL", "true");
        config.addDataSourceProperty("serverTimezone", "UTC");
        
        // 设置连接有效期小于IAM令牌有效期（15分钟）
        config.setMaxLifetime(10 * 60 * 1000); // 10分钟
        
        return new HikariDataSource(config);
    }
    
    @Bean
    public JdbcTemplate jdbcTemplate(DataSource dataSource) {
        return new JdbcTemplate(dataSource);
    }
}

@Repository
public class JdbcOrderRepository {
    
    private final JdbcTemplate