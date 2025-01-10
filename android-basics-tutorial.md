# Day 76: Java安卓开发 - 基础概念

## 1. 引言

在Android开发的学习过程中，掌握基础概念至关重要。本文将介绍Android开发中的核心概念，并通过实际的示例来加深理解。

## 2. Android应用的基本组件

### 2.1 Activity（活动）

Activity是Android应用程序的基本组成部分，代表着一个具有用户界面的单一屏幕。

#### 基本概念
- Activity本质上是一个Java类，继承自`android.app.Activity`
- 每个Activity都需要在AndroidManifest.xml中注册
- Activity具有生命周期，由系统管理

#### 示例代码

```java
public class MainActivity extends AppCompatActivity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        
        // 初始化UI组件
        Button btnClick = findViewById(R.id.btnClick);
        btnClick.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                Toast.makeText(MainActivity.this, "按钮被点击", Toast.LENGTH_SHORT).show();
            }
        });
    }
}
```

### 2.2 Service（服务）

Service是一个可以在后台执行长时间运行操作的应用组件，不提供用户界面。

#### 示例代码

```java
public class MyService extends Service {
    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }

    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        // 执行后台任务
        new Thread(new Runnable() {
            @Override
            public void run() {
                // 执行耗时操作
            }
        }).start();
        return START_STICKY;
    }
}
```

## 3. 布局与界面设计

### 3.1 常用布局

Android提供了多种布局方式：
- LinearLayout（线性布局）
- RelativeLayout（相对布局）
- ConstraintLayout（约束布局）
- FrameLayout（帧布局）

#### LinearLayout示例

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical">

    <TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Hello Android!"
        android:textSize="20sp"/>

    <Button
        android:id="@+id/btnClick"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="点击我"/>

</LinearLayout>
```

## 4. 实战Demo：简单计数器应用

让我们通过一个简单的计数器应用来综合运用以上概念。

### 4.1 布局文件 (activity_counter.xml)

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:gravity="center">

    <TextView
        android:id="@+id/tvCount"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="0"
        android:textSize="40sp"
        android:layout_marginBottom="20dp"/>

    <Button
        android:id="@+id/btnIncrement"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="增加"/>

    <Button
        android:id="@+id/btnDecrement"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="减少"/>

</LinearLayout>
```

### 4.2 Activity代码 (CounterActivity.java)

```java
public class CounterActivity extends AppCompatActivity {
    private TextView tvCount;
    private Button btnIncrement;
    private Button btnDecrement;
    private int count = 0;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_counter);

        // 初始化视图
        tvCount = findViewById(R.id.tvCount);
        btnIncrement = findViewById(R.id.btnIncrement);
        btnDecrement = findViewById(R.id.btnDecrement);

        // 设置点击事件
        btnIncrement.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                count++;
                updateCount();
            }
        });

        btnDecrement.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                count--;
                updateCount();
            }
        });
    }

    private void updateCount() {
        tvCount.setText(String.valueOf(count));
    }
}
```

### 4.3 在AndroidManifest.xml中注册

```xml
<activity
    android:name=".CounterActivity"
    android:exported="true">
    <intent-filter>
        <action android:name="android.intent.action.MAIN" />
        <category android:name="android.intent.category.LAUNCHER" />
    </intent-filter>
</activity>
```

## 5. 重要概念说明

### 5.1 生命周期

Activity的生命周期包含以下主要方法：
- onCreate(): 创建Activity时调用
- onStart(): Activity变为可见时调用
- onResume(): Activity可与用户交互时调用
- onPause(): Activity被暂停时调用
- onStop(): Activity不可见时调用
- onDestroy(): Activity被销毁时调用

### 5.2 Intent（意图）

Intent用于在不同的应用组件之间传递消息，是Android中重要的通信机制。

示例：启动新的Activity

```java
Intent intent = new Intent(MainActivity.this, SecondActivity.class);
intent.putExtra("key", "value"); // 传递数据
startActivity(intent);
```

## 6. 调试技巧

### 6.1 使用Logcat

```java
// 在代码中添加日志
Log.d("TAG", "调试信息");
Log.i("TAG", "信息");
Log.e("TAG", "错误信息");
```

### 6.2 使用断点调试
- 在Android Studio中点击行号设置断点
- 使用Debug模式运行应用
- 使用调试工具观察变量值和程序执行流程

## 7. 总结

本文介绍了Android开发的基础概念，包括：
- Activity的基本使用
- 服务的创建和生命周期
- 布局的设计和实现
- 简单的用户交互实现
- 调试技巧

通过计数器Demo的实现，我们实践了这些基础概念，为进一步学习Android开发打下了基础。

## 8. 练习建议

1. 尝试为计数器添加新功能，如重置按钮
2. 尝试保存计数值，使应用重启后保持原值
3. 尝试添加更多的界面元素和交互功能
4. 实践不同的布局方式
5. 熟悉Activity的生命周期方法

记住，实践是学习编程最好的方式，多动手写代码，多尝试新功能的实现。
