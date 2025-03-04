# Day 79: Java安卓开发 - 传感器应用

## 1. 引言

现代智能手机配备了各种传感器，使应用能够感知和响应物理环境。本文将介绍Android中的传感器框架，以及如何在应用中使用各种传感器。

## 2. Android传感器框架

Android提供了一套完整的传感器框架，允许开发者访问设备上的各种传感器。

### 2.1 传感器类型

Android支持三类传感器：

1. **运动传感器**：测量加速度和旋转力
   - 加速度传感器
   - 陀螺仪
   - 重力传感器

2. **环境传感器**：测量环境参数
   - 光线传感器
   - 温度传感器
   - 湿度传感器
   - 气压传感器

3. **位置传感器**：确定设备的物理位置
   - 磁力计（电子罗盘）
   - 接近传感器

### 2.2 传感器框架组件

- **SensorManager**：用于访问和管理传感器
- **Sensor**：表示特定的传感器
- **SensorEvent**：包含传感器报告的数据
- **SensorEventListener**：接收传感器数据变化的通知

## 3. 使用传感器的基本步骤

### 3.1 获取SensorManager

```java
private SensorManager sensorManager;

@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_sensor_demo);
    
    // 获取传感器管理器
    sensorManager = (SensorManager) getSystemService(Context.SENSOR_SERVICE);
}
```

### 3.2 访问传感器

```java
// 获取特定类型的传感器
Sensor accelerometer = sensorManager.getDefaultSensor(Sensor.TYPE_ACCELEROMETER);

// 检查设备是否有该传感器
if (accelerometer != null) {
    // 设备有加速度传感器
} else {
    // 设备没有加速度传感器
    Toast.makeText(this, "设备不支持加速度传感器", Toast.LENGTH_SHORT).show();
}

// 获取设备上所有可用的传感器
List<Sensor> deviceSensors = sensorManager.getSensorList(Sensor.TYPE_ALL);
for (Sensor sensor : deviceSensors) {
    Log.d("SensorDemo", "传感器名称: " + sensor.getName() + ", 类型: " + sensor.getType());
}
```

### 3.3 监听传感器数据

```java
public class SensorDemoActivity extends AppCompatActivity implements SensorEventListener {
    private SensorManager sensorManager;
    private Sensor accelerometer;
    private TextView tvAccelerometer;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_sensor_demo);
        
        tvAccelerometer = findViewById(R.id.tvAccelerometer);
        
        // 获取传感器管理器和加速度传感器
        sensorManager = (SensorManager) getSystemService(Context.SENSOR_SERVICE);
        accelerometer = sensorManager.getDefaultSensor(Sensor.TYPE_ACCELEROMETER);
    }

    @Override
    protected void onResume() {
        super.onResume();
        // 注册传感器监听器
        if (accelerometer != null) {
            sensorManager.registerListener(this, accelerometer, SensorManager.SENSOR_DELAY_NORMAL);
        }
    }

    @Override
    protected void onPause() {
        super.onPause();
        // 取消注册传感器监听器
        sensorManager.unregisterListener(this);
    }

    @Override
    public void onSensorChanged(SensorEvent event) {
        // 处理传感器数据
        if (event.sensor.getType() == Sensor.TYPE_ACCELEROMETER) {
            float x = event.values[0];
            float y = event.values[1];
            float z = event.values[2];
            
            String data = String.format("加速度传感器:\nX: %.2f\nY: %.2f\nZ: %.2f", x, y, z);
            tvAccelerometer.setText(data);
        }
    }

    @Override
    public void onAccuracyChanged(Sensor sensor, int accuracy) {
        // 传感器精度变化时调用
    }
}
```

## 4. 常用传感器示例

### 4.1 光线传感器

```java
public class LightSensorActivity extends AppCompatActivity implements SensorEventListener {
    private SensorManager sensorManager;
    private Sensor lightSensor;
    private TextView tvLightValue;
    private View rootView;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_light_sensor);
        
        tvLightValue = findViewById(R.id.tvLightValue);
        rootView = findViewById(R.id.rootView);
        
        sensorManager = (SensorManager) getSystemService(Context.SENSOR_SERVICE);
        lightSensor = sensorManager.getDefaultSensor(Sensor.TYPE_LIGHT);
        
        if (lightSensor == null) {
            Toast.makeText(this, "设备不支持光线传感器", Toast.LENGTH_SHORT).show();
            finish();
        }
    }

    @Override
    protected void onResume() {
        super.onResume();
        sensorManager.registerListener(this, lightSensor, SensorManager.SENSOR_DELAY_NORMAL);
    }

    @Override
    protected void onPause() {
        super.onPause();
        sensorManager.unregisterListener(this);
    }

    @Override
    public void onSensorChanged(SensorEvent event) {
        if (event.sensor.getType() == Sensor.TYPE_LIGHT) {
            float lightValue = event.values[0];
            tvLightValue.setText(String.format("光线强度: %.2f lux", lightValue));
            
            // 根据光线强度调整界面亮度
            if (lightValue < 10) {
                // 暗光环境
                rootView.setBackgroundColor(Color.DKGRAY);
                tvLightValue.setTextColor(Color.WHITE);
            } else {
                // 明亮环境
                rootView.setBackgroundColor(Color.WHITE);
                tvLightValue.setTextColor(Color.BLACK);
            }
        }
    }

    @Override
    public void onAccuracyChanged(Sensor sensor, int accuracy) {
        // 传感器精度变化
    }
}
```

### 4.2 接近传感器

```java
public class ProximitySensorActivity extends AppCompatActivity implements SensorEventListener {
    private SensorManager sensorManager;
    private Sensor proximitySensor;
    private TextView tvProximity;
    private ImageView ivProximity;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_proximity_sensor);
        
        tvProximity = findViewById(R.id.tvProximity);
        ivProximity = findViewById(R.id.ivProximity);
        
        sensorManager = (SensorManager) getSystemService(Context.SENSOR_SERVICE);
        proximitySensor = sensorManager.getDefaultSensor(Sensor.TYPE_PROXIMITY);
        
        if (proximitySensor == null) {
            Toast.makeText(this, "设备不支持接近传感器", Toast.LENGTH_SHORT).show();
            finish();
        }
    }

    @Override
    protected void onResume() {
        super.onResume();
        sensorManager.registerListener(this, proximitySensor, SensorManager.SENSOR_DELAY_NORMAL);
    }

    @Override
    protected void onPause() {
        super.onPause();
        sensorManager.unregisterListener(this);
    }

    @Override
    public void onSensorChanged(SensorEvent event) {
        if (event.sensor.getType() == Sensor.TYPE_PROXIMITY) {
            float distance = event.values[0];
            float maxRange = proximitySensor.getMaximumRange();
            
            if (distance < maxRange) {
                // 物体靠近
                tvProximity.setText("检测到物体靠近!");
                ivProximity.setImageResource(R.drawable.ic_near); // 替换为实际资源
            } else {
                // 没有物体靠近
                tvProximity.setText("未检测到物体靠近");
                ivProximity.setImageResource(R.drawable.ic_far); // 替换为实际资源
            }
        }
    }

    @Override
    public void onAccuracyChanged(Sensor sensor, int accuracy) {
        // 传感器精度变化
    }
}
```

### 4.3 陀螺仪传感器

```java
public class GyroscopeActivity extends AppCompatActivity implements SensorEventListener {
    private SensorManager sensorManager;
    private Sensor gyroscope;
    private TextView tvGyroscope;
    private long lastUpdate = 0;
    private float[] rotationMatrix = new float[9];
    private float[] orientationValues = new float[3];

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_gyroscope);
        
        tvGyroscope = findViewById(R.id.tvGyroscope);
        
        sensorManager = (SensorManager) getSystemService(Context.SENSOR_SERVICE);
        gyroscope = sensorManager.getDefaultSensor(Sensor.TYPE_GYROSCOPE);
        
        if (gyroscope == null) {
            Toast.makeText(this, "设备不支持陀螺仪传感器", Toast.LENGTH_SHORT).show();
            finish();
        }
    }

    @Override
    protected void onResume() {
        super.onResume();
        sensorManager.registerListener(this, gyroscope, SensorManager.SENSOR_DELAY_GAME);
    }

    @Override
    protected void onPause() {
        super.onPause();
        sensorManager.unregisterListener(this);
    }

    @Override
    public void onSensorChanged(SensorEvent event) {
        if (event.sensor.getType() == Sensor.TYPE_GYROSCOPE) {
            long curTime = System.currentTimeMillis();
            
            // 限制更新频率
            if ((curTime - lastUpdate) > 100) {
                lastUpdate = curTime;
                
                float x = event.values[0]; // 绕x轴旋转的角速度
                float y = event.values[1]; // 绕y轴旋转的角速度
                float z = event.values[2]; // 绕z轴旋转的角速度
                
                String data = String.format("陀螺仪数据:\n绕X轴旋转: %.2f rad/s\n绕Y轴旋转: %.2f rad/s\n绕Z轴旋转: %.2f rad/s", x, y, z);
                tvGyroscope.setText(data);
                
                // 检测设备旋转
                detectRotation(x, y, z);
            }
        }
    }

    private void detectRotation(float x, float y, float z) {
        // 简单的旋转检测
        float threshold = 0.5f; // 阈值
        
        if (Math.abs(x) > threshold) {
            if (x > 0) {
                Toast.makeText(this, "设备向右旋转", Toast.LENGTH_SHORT).show();
            } else {
                Toast.makeText(this, "设备向左旋转", Toast.LENGTH_SHORT).show();
            }
        }
        
        if (Math.abs(y) > threshold) {
            if (y > 0) {
                Toast.makeText(this, "设备向上旋转", Toast.LENGTH_SHORT).show();
            } else {
                Toast.makeText(this, "设备向下旋转", Toast.LENGTH_SHORT).show();
            }
        }
    }

    @Override
    public void onAccuracyChanged(Sensor sensor, int accuracy) {
        // 传感器精度变化
    }
}
```

## 5. 传感器融合

有时候，单个传感器的数据可能不够准确或完整，可以通过融合多个传感器的数据来获得更准确的结果。

### 5.1 方向传感器示例

```java
public class OrientationActivity extends AppCompatActivity implements SensorEventListener {
    private SensorManager sensorManager;
    private Sensor accelerometer;
    private Sensor magnetometer;
    private TextView tvOrientation;
    private float[] accelerometerValues = new float[3];
    private float[] magnetometerValues = new float[3];
    private float[] rotationMatrix = new float[9];
    private float[] orientationValues = new float[3];

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_orientation);
        
        tvOrientation = findViewById(R.id.tvOrientation);
        
        sensorManager = (SensorManager) getSystemService(Context.SENSOR_SERVICE);
        accelerometer = sensorManager.getDefaultSensor(Sensor.TYPE_ACCELEROMETER);
        magnetometer = sensorManager.getDefaultSensor(Sensor.TYPE_MAGNETIC_FIELD);
        
        if (accelerometer == null || magnetometer == null) {
            Toast.makeText(this, "设备不支持所需传感器", Toast.LENGTH_SHORT).show();
            finish();
        }
    }

    @Override
    protected void onResume() {
        super.onResume();
        sensorManager.registerListener(this, accelerometer, SensorManager.SENSOR_DELAY_NORMAL);
        sensorManager.registerListener(this, magnetometer, SensorManager.SENSOR_DELAY_NORMAL);
    }

    @Override
    protected void onPause() {
        super.onPause();
        sensorManager.unregisterListener(this);
    }

    @Override
    public void onSensorChanged(SensorEvent event) {
        if (event.sensor.getType() == Sensor.TYPE_ACCELEROMETER) {
            System.arraycopy(event.values, 0, accelerometerValues, 0, 3);
        } else if (event.sensor.getType() == Sensor.TYPE_MAGNETIC_FIELD) {
            System.arraycopy(event.values, 0, magnetometerValues, 0, 3);
        }
        
        // 当两个传感器都有数据时，计算方向
        if (accelerometerValues != null && magnetometerValues != null) {
            boolean success = SensorManager.getRotationMatrix(rotationMatrix, null, 
                    accelerometerValues, magnetometerValues);
            
            if (success) {
                SensorManager.getOrientation(rotationMatrix, orientationValues);
                
                // 将弧度转换为角度
                float azimuth = (float) Math.toDegrees(orientationValues[0]); // 方位角
                float pitch = (float) Math.toDegrees(orientationValues[1]);   // 俯仰角
                float roll = (float) Math.toDegrees(orientationValues[2]);    // 滚转角
                
                // 确保方位角在0-360度之间
                if (azimuth < 0) {
                    azimuth += 360;
                }
                
                String direction = getDirection(azimuth);
                String data = String.format("方位角: %.1f° (%s)\n俯仰角: %.1f°\n滚转角: %.1f°", 
                        azimuth, direction, pitch, roll);
                tvOrientation.setText(data);
            }
        }
    }
    
    private String getDirection(float azimuth) {
        if (azimuth >= 337.5 || azimuth < 22.5) {
            return "北";
        } else if (azimuth >= 22.5 && azimuth < 67.5) {
            return "东北";
        } else if (azimuth >= 67.5 && azimuth < 112.5) {
            return "东";
        } else if (azimuth >= 112.5 && azimuth < 157.5) {
            return "东南";
        } else if (azimuth >= 157.5 && azimuth < 202.5) {
            return "南";
        } else if (azimuth >= 202.5 && azimuth < 247.5) {
            return "西南";
        } else if (azimuth >= 247.5 && azimuth < 292.5) {
            return "西";
        } else {
            return "西北";
        }
    }

    @Override
    public void onAccuracyChanged(Sensor sensor, int accuracy) {
        // 传感器精度变化
    }
}
```

## 6. 实用传感器应用场景

### 6.1 计步器

```java
public class StepCounterActivity extends AppCompatActivity implements SensorEventListener {
    private SensorManager sensorManager;
    private Sensor stepCounter;
    private TextView tvStepCount;
    private boolean isStepCounterSupported;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_step_counter);
        
        tvStepCount = findViewById(R.id.tvStepCount);
        
        sensorManager = (SensorManager) getSystemService(Context.SENSOR_SERVICE);
        stepCounter = sensorManager.getDefaultSensor(Sensor.TYPE_STEP_COUNTER);
        
        isStepCounterSupported = (stepCounter != null);
        
        if (!isStepCounterSupported) {
            tvStepCount.setText("设备不支持计步器传感器");
        }
    }

    @Override
    protected void onResume() {
        super.onResume();
        if (isStepCounterSupported) {
            sensorManager.registerListener(this, stepCounter, SensorManager.SENSOR_DELAY_NORMAL);
        }
    }

    @Override
    protected void onPause() {
        super.onPause();
        if (isStepCounterSupported) {
            sensorManager.unregisterListener(this);
        }
    }

    @Override
    public void onSensorChanged(SensorEvent event) {
        if (event.sensor.getType() == Sensor.TYPE_STEP_COUNTER) {
            int steps = (int) event.values[0];
            tvStepCount.setText("步数: " + steps);
        }
    }

    @Override
    public void onAccuracyChanged(Sensor sensor, int accuracy) {
        // 传感器精度变化
    }
}
```

### 6.2 摇一摇功能

```java
public class ShakeActivity extends AppCompatActivity implements SensorEventListener {
    private SensorManager sensorManager;
    private Sensor accelerometer;
    private TextView tvShakeStatus;
    
    private static final float SHAKE_THRESHOLD = 12.0f;
    private static final int MIN_TIME_BETWEEN_SHAKES = 1000; // 毫秒
    private long lastShakeTime = 0;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_shake);
        
        tvShakeStatus = findViewById(R.id.tvShakeStatus);
        
        sensorManager = (SensorManager) getSystemService(Context.SENSOR_SERVICE);
        accelerometer = sensorManager.getDefaultSensor(Sensor.TYPE_ACCELEROMETER);
    }

    @Override
    protected void onResume() {
        super.onResume();
        sensorManager.registerListener(this, accelerometer, SensorManager.SENSOR_DELAY_NORMAL);
    }

    @Override
    protected void onPause() {
        super.onPause();
        sensorManager.unregisterListener(this);
    }

    @Override
    public void onSensorChanged(SensorEvent event) {
        if (event.sensor.getType() == Sensor.TYPE_ACCELEROMETER) {
            float x = event.values[0];
            float y = event.values[1];
            float z = event.values[2];
            
            float acceleration = (float) Math.sqrt(x*x + y*y + z*z) - SensorManager.GRAVITY_EARTH;
            
            if (acceleration > SHAKE_THRESHOLD) {
                long currentTime = System.currentTimeMillis();
                if (currentTime - lastShakeTime > MIN_TIME_BETWEEN_SHAKES) {
                    lastShakeTime = currentTime;
                    onShakeDetected();
                }
            }
        }
    }
    
    private void onShakeDetected() {
        tvShakeStatus.setText("检测到摇晃！" + System.currentTimeMillis());
        // 可以添加震动、声音等反馈
        Vibrator vibrator = (Vibrator) getSystemService(Context.VIBRATOR_SERVICE);
        if (vibrator.hasVibrator()) {
            vibrator.vibrate(500); // 震动500毫秒
        }
    }

    @Override
    public void onAccuracyChanged(Sensor sensor, int accuracy) {
        // 传感器精度变化
    }
}
```

## 7. 总结

本文介绍了Android传感器框架的基本概念和使用方法：

1. 了解了Android支持的三类传感器：运动传感器、环境传感器和位置传感器
2. 学习了如何获取和使用各种传感器，包括加速度传感器、光线传感器、接近传感器和陀螺仪
3. 掌握了传感器数据的处理和应用，如方向检测、计步器和摇一摇功能

通过合理使用传感器，可以为应用增加更多交互方式，提升用户体验。在实际开发中，需要注意以下几点：

- 不是所有设备都支持所有传感器，使用前应进行检查
- 传感器监听应在适当的生命周期方法中注册和注销
- 传感器数据可能需要滤波处理以减少噪声
- 频繁的传感器数据更新可能会消耗电量，应根据需要调整更新频率

## 8. 练习任务

1. 开发一个指南针应用，使用方向传感器显示方向
2. 实现一个水平仪应用，使用加速度传感器检测设备是否水平
3. 创建一个光线检测器，根据环境光线强度自动调整应用界面亮度
4. 开发一个简单的健身应用，使用计步器传感器记录用户的步数