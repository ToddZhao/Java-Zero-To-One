# Day 124: Java架构设计 - 微内核架构

## 1. 微内核架构概述

微内核架构（Microkernel Architecture）是一种软件架构模式，它将系统的核心功能与扩展功能分离。核心系统（微内核）提供最小化的功能集，而额外的功能则通过插件或扩展模块实现。这种架构模式特别适合需要高度可扩展性和可定制性的系统。

## 2. 微内核架构核心概念

### 2.1 核心系统（微内核）

```java
// 插件接口
public interface Plugin {
    String getName();
    String getVersion();
    void initialize();
    void shutdown();
}

// 插件管理器
public class PluginManager {
    private final Map<String, Plugin> plugins = new HashMap<>();
    
    public void registerPlugin(Plugin plugin) {
        if (plugins.containsKey(plugin.getName())) {
            throw new PluginAlreadyRegisteredException(plugin.getName());
        }
        
        plugins.put(plugin.getName(), plugin);
        plugin.initialize();
    }
    
    public void unregisterPlugin(String pluginName) {
        Plugin plugin = plugins.get(pluginName);
        if (plugin != null) {
            plugin.shutdown();
            plugins.remove(pluginName);
        }
    }
    
    public Plugin getPlugin(String pluginName) {
        return plugins.get(pluginName);
    }
    
    public Collection<Plugin> getAllPlugins() {
        return Collections.unmodifiableCollection(plugins.values());
    }
}

// 核心系统
public class CoreSystem {
    private final PluginManager pluginManager;
    private final EventBus eventBus;
    
    public CoreSystem() {
        this.pluginManager = new PluginManager();
        this.eventBus = new EventBus();
    }
    
    public void start() {
        // 初始化核心系统
        System.out.println("Core system starting...");
        
        // 加载插件
        loadPlugins();
        
        System.out.println("Core system started");
    }
    
    public void stop() {
        // 关闭所有插件
        for (Plugin plugin : pluginManager.getAllPlugins()) {
            pluginManager.unregisterPlugin(plugin.getName());
        }
        
        System.out.println("Core system stopped");
    }
    
    private void loadPlugins() {
        // 从配置或插件目录加载插件
        ServiceLoader<Plugin> loader = ServiceLoader.load(Plugin.class);
        for (Plugin plugin : loader) {
            pluginManager.registerPlugin(plugin);
        }
    }
    
    public PluginManager getPluginManager() {
        return pluginManager;
    }
    
    public EventBus getEventBus() {
        return eventBus;
    }
}
```

### 2.2 插件实现

```java
// 事件总线
public class EventBus {
    private final Map<String, List<EventListener>> listeners = new HashMap<>();
    
    public void subscribe(String eventType, EventListener listener) {
        listeners.computeIfAbsent(eventType, k -> new ArrayList<>()).add(listener);
    }
    
    public void unsubscribe(String eventType, EventListener listener) {
        if (listeners.containsKey(eventType)) {
            listeners.get(eventType).remove(listener);
        }
    }
    
    public void publish(Event event) {
        List<EventListener> eventListeners = listeners.getOrDefault(event.getType(), Collections.emptyList());
        for (EventListener listener : eventListeners) {
            listener.onEvent(event);
        }
    }
}

// 事件接口
public interface Event {
    String getType();
    Map<String, Object> getData();
}

// 事件监听器接口
public interface EventListener {
    void onEvent(Event event);
}

// 具体插件实现
@AutoService(Plugin.class) // 使用Google AutoService进行SPI注册
public class PaymentPlugin implements Plugin {
    private EventBus eventBus;
    
    @Override
    public String getName() {
        return "payment-plugin";
    }
    
    @Override
    public String getVersion() {
        return "1.0.0";
    }
    
    @Override
    public void initialize() {
        System.out.println("Payment plugin initializing...");
        
        // 获取核心系统的事件总线
        CoreSystem coreSystem = CoreSystem.getInstance();
        this.eventBus = coreSystem.getEventBus();
        
        // 注册事件监听器
        eventBus.subscribe("order.created", this::handleOrderCreated);
        
        System.out.println("Payment plugin initialized");
    }
    
    @Override
    public void shutdown() {
        System.out.println("Payment plugin shutting down...");
        
        // 取消注册事件监听器
        if (eventBus != null) {
            eventBus.unsubscribe("order.created", this::handleOrderCreated);
        }
        
        System.out.println("Payment plugin shutdown");
    }
    
    private void handleOrderCreated(Event event) {
        Map<String, Object> data = event.getData();
        String orderId = (String) data.get("orderId");
        BigDecimal amount = (BigDecimal) data.get("amount");
        
        System.out.println("Processing payment for order: " + orderId + ", amount: " + amount);
        
        // 处理支付逻辑
        boolean paymentSuccessful = processPayment(orderId, amount);
        
        // 发布支付结果事件
        Map<String, Object> paymentData = new HashMap<>();
        paymentData.put("orderId", orderId);
        paymentData.put("successful", paymentSuccessful);
        
        Event paymentEvent = new SimpleEvent("payment.processed", paymentData);
        eventBus.publish(paymentEvent);
    }
    
    private boolean processPayment(String orderId, BigDecimal amount) {
        // 实际支付处理逻辑
        return true; // 假设支付成功
    }
}
```

## 3. 插件加载与生命周期管理

```java
// 插件加载器
public class PluginLoader {
    private final Path pluginsDirectory;
    
    public PluginLoader(Path pluginsDirectory) {
        this.pluginsDirectory = pluginsDirectory;
    }
    
    public List<Plugin> loadPlugins() throws IOException {
        List<Plugin> loadedPlugins = new ArrayList<>();
        
        // 确保插件目录存在
        if (!Files.exists(pluginsDirectory)) {
            Files.createDirectories(pluginsDirectory);
            return loadedPlugins;
        }
        
        // 遍历插件目录中的所有JAR文件
        try (DirectoryStream<Path> stream = Files.newDirectoryStream(pluginsDirectory, "*.jar")) {
            for (Path jarPath : stream) {
                try {
                    List<Plugin> plugins = loadPluginsFromJar(jarPath);
                    loadedPlugins.addAll(plugins);
                } catch (Exception e) {
                    System.err.println("Failed to load plugins from " + jarPath + ": " + e.getMessage());
                }
            }
        }
        
        return loadedPlugins;
    }
    
    private List<Plugin> loadPluginsFromJar(Path jarPath) throws Exception {
        List<Plugin> plugins = new ArrayList<>();
        
        // 创建一个新的类加载器来加载插件JAR
        URL[] urls = new URL[] { jarPath.toUri().toURL() };
        try (URLClassLoader classLoader = new URLClassLoader(urls, getClass().getClassLoader())) {
            // 使用Java SPI机制加载插件
            ServiceLoader<Plugin> loader = ServiceLoader.load(Plugin.class, classLoader);
            for (Plugin plugin : loader) {
                plugins.add(plugin);
            }
        }
        
        return plugins;
    }
}

// 插件生命周期管理
public class PluginLifecycleManager {
    private final CoreSystem coreSystem;
    private final PluginLoader pluginLoader;
    
    public PluginLifecycleManager(CoreSystem coreSystem, Path pluginsDirectory) {
        this.coreSystem = coreSystem;
        this.pluginLoader = new PluginLoader(pluginsDirectory);
    }
    
    public void loadAndRegisterPlugins() throws IOException {
        List<Plugin> plugins = pluginLoader.loadPlugins();
        for (Plugin plugin : plugins) {
            try {
                coreSystem.getPluginManager().registerPlugin(plugin);
                System.out.println("Registered plugin: " + plugin.getName() + " v" + plugin.getVersion());
            } catch (Exception e) {
                System.err.println("Failed to register plugin " + plugin.getName() + ": " + e.getMessage());
            }
        }
    }
    
    public void reloadPlugins() throws IOException {
        // 卸载所有现有插件
        for (Plugin plugin : coreSystem.getPluginManager().getAllPlugins()) {
            coreSystem.getPluginManager().unregisterPlugin(plugin.getName());
        }
        
        // 重新加载插件
        loadAndRegisterPlugins();
    }
}
```

## 4. 微内核架构实践

### 4.1 应用示例

```java
public class MicrokernelApplication {
    public static void main(String[] args) throws Exception {
        // 创建并启动核心系统
        CoreSystem coreSystem = new CoreSystem();
        coreSystem.start();
        
        // 设置插件目录并加载插件
        Path pluginsDirectory = Paths.get("plugins");
        PluginLifecycleManager lifecycleManager = new PluginLifecycleManager(coreSystem, pluginsDirectory);
        lifecycleManager.loadAndRegisterPlugins();
        
        // 发布一个订单创建事件，测试插件功能
        Map<String, Object> orderData = new HashMap<>();
        orderData.put("orderId", "ORD-12345");
        orderData.put("amount", new BigDecimal("99.99"));
        
        Event orderCreatedEvent = new SimpleEvent("order.created", orderData);
        coreSystem.getEventBus().publish(orderCreatedEvent);
        
        // 等待一段时间，让插件处理事件
        Thread.sleep(1000);
        
        // 关闭系统
        coreSystem.stop();
    }
}

// 简单事件实现
public class SimpleEvent implements Event {
    private final String type;
    private final Map<String, Object> data;
    
    public SimpleEvent(String type, Map<String, Object> data) {
        this.type = type;
        this.data = Collections.unmodifiableMap(new HashMap<>(data));
    }
    
    @Override
    public String getType() {
        return type;
    }
    
    @Override
    public Map<String, Object> getData() {
        return data;
    }
}
```

### 4.2 插件配置

```java
// 插件配置接口
public interface PluginConfig {
    String getProperty(String key);
    String getProperty(String key, String defaultValue);
    int getIntProperty(String key, int defaultValue);
    boolean getBooleanProperty(String key, boolean defaultValue);
    void setProperty(String key, String value);
}

// 基于属性文件的插件配置实现
public class PropertiesPluginConfig implements PluginConfig {
    private final Properties properties = new Properties();
    private final Path configFile;
    
    public PropertiesPluginConfig(Path configFile) {
        this.configFile = configFile;
        loadConfig();
    }
    
    private void loadConfig() {
        if (Files.exists(configFile)) {
            try (InputStream is = Files.newInputStream(configFile)) {
                properties.load(is);
            } catch (IOException e) {
                System.err.println("Failed to load plugin config: " + e.getMessage());
            }
        }
    }
    
    private void saveConfig() {
        try (OutputStream os = Files.newOutputStream(configFile)) {
            properties.store(os, "Plugin Configuration");
        } catch (IOException e) {
            System.err.println("Failed to save plugin config: " + e.getMessage());
        }
    }
    
    @Override
    public String getProperty(String key) {
        return properties.getProperty(key);
    }
    
    @Override
    public String getProperty(String key, String defaultValue) {
        return properties.getProperty(key, defaultValue);
    }
    
    @Override
    public int getIntProperty(String key, int defaultValue) {
        String value = getProperty(key);
        if (value == null) {
            return defaultValue;
        }
        try {
            return Integer.parseInt(value);
        } catch (NumberFormatException e) {
            return defaultValue;
        }
    }
    
    @Override
    public boolean getBooleanProperty(String key, boolean defaultValue) {
        String value = getProperty(key);
        if (value == null) {
            return defaultValue;
        }
        return Boolean.parseBoolean(value);