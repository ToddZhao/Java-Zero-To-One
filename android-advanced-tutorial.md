# Day 91: Java安卓开发 - 高级特性

## 1. 引言

在掌握了Android开发的基础概念后，我们需要进一步探索Android的高级特性，以构建功能更加丰富、用户体验更佳的应用程序。本文将介绍Android开发中的高级组件和技术。

## 2. 高级UI组件

### 2.1 RecyclerView

RecyclerView是ListView的更强大替代品，提供了更灵活的列表显示方式和更好的性能。

#### 基本实现

```java
public class MainActivity extends AppCompatActivity {
    private RecyclerView recyclerView;
    private MyAdapter adapter;
    private List<String> dataList;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        // 初始化数据
        dataList = new ArrayList<>();
        for (int i = 0; i < 50; i++) {
            dataList.add("Item " + i);
        }

        // 设置RecyclerView
        recyclerView = findViewById(R.id.recyclerView);
        recyclerView.setLayoutManager(new LinearLayoutManager(this));
        adapter = new MyAdapter(dataList);
        recyclerView.setAdapter(adapter);
    }
}
```

#### 自定义适配器

```java
public class MyAdapter extends RecyclerView.Adapter<MyAdapter.ViewHolder> {
    private List<String> dataList;

    public MyAdapter(List<String> dataList) {
        this.dataList = dataList;
    }

    @NonNull
    @Override
    public ViewHolder onCreateViewHolder(@NonNull ViewGroup parent, int viewType) {
        View view = LayoutInflater.from(parent.getContext())
                .inflate(R.layout.item_layout, parent, false);
        return new ViewHolder(view);
    }

    @Override
    public void onBindViewHolder(@NonNull ViewHolder holder, int position) {
        holder.textView.setText(dataList.get(position));
        holder.itemView.setOnClickListener(v -> {
            // 处理点击事件
            Toast.makeText(v.getContext(), "点击了: " + dataList.get(position), 
                    Toast.LENGTH_SHORT).show();
        });
    }

    @Override
    public int getItemCount() {
        return dataList.size();
    }

    public static class ViewHolder extends RecyclerView.ViewHolder {
        TextView textView;

        public ViewHolder(@NonNull View itemView) {
            super(itemView);
            textView = itemView.findViewById(R.id.textView);
        }
    }
}
```

### 2.2 ViewPager2

ViewPager2是ViewPager的升级版，用于实现滑动页面效果。

```java
public class MainActivity extends AppCompatActivity {
    private ViewPager2 viewPager;
    private TabLayout tabLayout;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        viewPager = findViewById(R.id.viewPager);
        tabLayout = findViewById(R.id.tabLayout);

        // 设置适配器
        FragmentStateAdapter pagerAdapter = new ScreenSlidePagerAdapter(this);
        viewPager.setAdapter(pagerAdapter);

        // 连接TabLayout和ViewPager2
        new TabLayoutMediator(tabLayout, viewPager,
                (tab, position) -> tab.setText("页面 " + (position + 1)))
                .attach();
    }

    private class ScreenSlidePagerAdapter extends FragmentStateAdapter {
        public ScreenSlidePagerAdapter(FragmentActivity fa) {
            super(fa);
        }

        @Override
        public Fragment createFragment(int position) {
            // 根据位置返回不同的Fragment
            return PageFragment.newInstance(position);
        }

        @Override
        public int getItemCount() {
            return 5; // 总页数
        }
    }
}
```

## 3. 网络操作

### 3.1 Retrofit使用

Retrofit是一个类型安全的HTTP客户端，简化了网络请求的实现。

#### 依赖配置

```gradle
dependencies {
    implementation 'com.squareup.retrofit2:retrofit:2.9.0'
    implementation 'com.squareup.retrofit2:converter-gson:2.9.0'
    implementation 'com.squareup.okhttp3:logging-interceptor:4.9.0'
}
```

#### API接口定义

```java
public interface ApiService {
    @GET("users/{user}/repos")
    Call<List<Repo>> listRepos(@Path("user") String user);

    @POST("users/new")
    Call<User> createUser(@Body User user);
}
```

#### 网络请求实现

```java
public class NetworkManager {
    private static final String BASE_URL = "https://api.github.com/";
    private static NetworkManager instance;
    private ApiService apiService;

    private NetworkManager() {
        // 创建OkHttpClient
        OkHttpClient client = new OkHttpClient.Builder()
                .addInterceptor(new HttpLoggingInterceptor().setLevel(HttpLoggingInterceptor.Level.BODY))
                .build();

        // 创建Retrofit实例
        Retrofit retrofit = new Retrofit.Builder()
                .baseUrl(BASE_URL)
                .client(client)
                .addConverterFactory(GsonConverterFactory.create())
                .build();

        // 创建API服务
        apiService = retrofit.create(ApiService.class);
    }

    public static synchronized NetworkManager getInstance() {
        if (instance == null) {
            instance = new NetworkManager();
        }
        return instance;
    }

    public void getRepositories(String user, Callback<List<Repo>> callback) {
        Call<List<Repo>> call = apiService.listRepos(user);
        call.enqueue(callback);
    }
}
```

### 3.2 使用示例

```java
public class MainActivity extends AppCompatActivity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        Button btnFetch = findViewById(R.id.btnFetch);
        TextView tvResult = findViewById(R.id.tvResult);

        btnFetch.setOnClickListener(v -> {
            NetworkManager.getInstance().getRepositories("octocat", new Callback<List<Repo>>() {
                @Override
                public void onResponse(Call<List<Repo>> call, Response<List<Repo>> response) {
                    if (response.isSuccessful() && response.body() != null) {
                        StringBuilder result = new StringBuilder();
                        for (Repo repo : response.body()) {
                            result.append(repo.getName()).append("\n");
                        }
                        tvResult.setText(result.toString());
                    }
                }

                @Override
                public void onFailure(Call<List<Repo>> call, Throwable t) {
                    tvResult.setText("请求失败: " + t.getMessage());
                }
            });
        });
    }
}
```

## 4. 传感器应用

### 4.1 加速度传感器

```java
public class SensorActivity extends AppCompatActivity implements SensorEventListener {
    private SensorManager sensorManager;
    private Sensor accelerometer;
    private TextView tvX, tvY, tvZ;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_sensor);

        // 初始化视图
        tvX = findViewById(R.id.tvX);
        tvY = findViewById(R.id.tvY);
        tvZ = findViewById(R.id.tvZ);

        // 获取传感器服务
        sensorManager = (SensorManager) getSystemService(Context.SENSOR_SERVICE);
        if (sensorManager != null) {
            accelerometer = sensorManager.getDefaultSensor(Sensor.TYPE_ACCELEROMETER);
        }
    }

    @Override
    protected void onResume() {
        super.onResume();
        if (accelerometer != null) {
            sensorManager.registerListener(this, accelerometer, SensorManager.SENSOR_DELAY_NORMAL);
        }
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

            tvX.setText("X轴: " + x);
            tvY.setText("Y轴: " + y);
            tvZ.setText("Z轴: " + z);
        }
    }

    @Override
    public void onAccuracyChanged(Sensor sensor, int accuracy) {
        // 传感器精度变化时的处理
    }
}
```

### 4.2 位置传感器

```java
public class LocationActivity extends AppCompatActivity {
    private FusedLocationProviderClient fusedLocationClient;
    private TextView tvLocation;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_location);

        tvLocation = findViewById(R.id.tvLocation);
        fusedLocationClient = LocationServices.getFusedLocationProviderClient(this);

        Button btnGetLocation = findViewById(R.id.btnGetLocation);
        btnGetLocation.setOnClickListener(v -> getLastLocation());
    }

    private void getLastLocation() {
        if (ActivityCompat.checkSelfPermission(this, Manifest.permission.ACCESS_FINE_LOCATION) 
                != PackageManager.PERMISSION_GRANTED) {
            ActivityCompat.requestPermissions(this, 
                    new String[]{Manifest.permission.ACCESS_FINE_LOCATION}, 1);
            return;
        }

        fusedLocationClient.getLastLocation()
                .addOnSuccessListener(this, location -> {
                    if (location != null) {
                        tvLocation.setText("纬度: " + location.getLatitude() + 
                                "\n经度: " + location.getLongitude());
                    } else {
                        tvLocation.setText("无法获取位置信息");
                    }
                });
    }

    @Override
    public void onRequestPermissionsResult(int requestCode, @NonNull String[] permissions, 
                                           @NonNull int[] grantResults) {
        super.onRequestPermissionsResult(requestCode, permissions, grantResults);
        if (requestCode == 1 && grantResults.length > 0 
                && grantResults[0] == PackageManager.PERMISSION_GRANTED) {
            getLastLocation();
        }
    }
}
```

## 5. 数据存储

### 5.1 Room数据库

Room是Android官方推荐的本地数据库解决方案。

#### 依赖配置

```gradle
dependencies {
    implementation "androidx.room:room-runtime:2.4.2"
    annotationProcessor "androidx.room:room-compiler:2.4.2"
}
```

#### 实体类定义

```java
@Entity(tableName = "users")
public class User {
    @PrimaryKey(autoGenerate = true)
    private int id;

    @ColumnInfo(name = "name")
    private String name;

    @ColumnInfo(name = "age")
    private int age;

    // 构造函数、getter和setter
    public User(String name, int age) {
        this.name = name;
        this.age = age;
    }

    public int getId() { return id; }
    public void setId(int id) { this.id = id; }
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public int getAge() { return age; }
    public void setAge(int age) { this.age = age; }
}
```

#### DAO接口

```java
@Dao
public interface UserDao {
    @Query("SELECT * FROM users")
    List<User> getAll();

    @Query("SELECT * FROM users WHERE id = :userId")
    User getById(int userId);

    @Insert
    void insert(User user);

    @Update
    void update(User user);

    @Delete
    void delete(User user);
}
```

#### 数据库类

```java
@Database(entities = {User.class}, version = 1)
public abstract class AppDatabase extends RoomDatabase {
    private static AppDatabase instance;

    public abstract UserDao userDao();

    public static synchronized AppDatabase getInstance(Context context) {
        if (instance == null) {
            instance = Room.databaseBuilder(context.getApplicationContext(),
                    AppDatabase.class, "app_database")
                    .fallbackToDestructiveMigration()
                    .build();
        }
        return instance;
    }
}
```

### 5.2 使用示例

```