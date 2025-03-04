# Day 78: Java安卓开发 - 网络编程

## 1. 引言

网络通信是现代移动应用的核心功能之一。本文将详细介绍Android中的网络编程技术，包括基本的HTTP请求、常用网络库以及WebSocket通信等内容。

## 2. 网络权限配置

在进行网络操作前，需要在AndroidManifest.xml中添加网络权限：

```xml
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
```

## 3. 基本HTTP请求

### 3.1 使用HttpURLConnection

```java
public class HttpDemoActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_http_demo);

        Button btnFetch = findViewById(R.id.btnFetch);
        btnFetch.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                new FetchDataTask().execute("https://jsonplaceholder.typicode.com/posts/1");
            }
        });
    }

    private class FetchDataTask extends AsyncTask<String, Void, String> {

        @Override
        protected String doInBackground(String... urls) {
            try {
                URL url = new URL(urls[0]);
                HttpURLConnection connection = (HttpURLConnection) url.openConnection();
                connection.setRequestMethod("GET");
                connection.setConnectTimeout(5000);
                connection.setReadTimeout(5000);

                int responseCode = connection.getResponseCode();
                if (responseCode == HttpURLConnection.HTTP_OK) {
                    BufferedReader reader = new BufferedReader(
                            new InputStreamReader(connection.getInputStream()));
                    StringBuilder response = new StringBuilder();
                    String line;
                    while ((line = reader.readLine()) != null) {
                        response.append(line);
                    }
                    reader.close();
                    return response.toString();
                } else {
                    return "Error: " + responseCode;
                }
            } catch (Exception e) {
                return "Error: " + e.getMessage();
            }
        }

        @Override
        protected void onPostExecute(String result) {
            Toast.makeText(HttpDemoActivity.this, result, Toast.LENGTH_LONG).show();
            Log.d("HttpDemo", result);
        }
    }
}
```

### 3.2 网络操作的最佳实践

1. 不要在主线程中进行网络操作
2. 添加超时设置
3. 处理网络异常
4. 检查网络连接状态

```java
private boolean isNetworkAvailable() {
    ConnectivityManager connectivityManager = (ConnectivityManager) getSystemService(Context.CONNECTIVITY_SERVICE);
    NetworkInfo activeNetworkInfo = connectivityManager.getActiveNetworkInfo();
    return activeNetworkInfo != null && activeNetworkInfo.isConnected();
}
```

## 4. 使用Retrofit库

Retrofit是一个类型安全的HTTP客户端，使网络请求更加简洁和高效。

### 4.1 添加依赖

在build.gradle文件中添加：

```gradle
dependencies {
    implementation 'com.squareup.retrofit2:retrofit:2.9.0'
    implementation 'com.squareup.retrofit2:converter-gson:2.9.0'
    implementation 'com.squareup.okhttp3:logging-interceptor:4.9.0'
}
```

### 4.2 创建API接口

```java
public interface ApiService {
    @GET("posts/{id}")
    Call<Post> getPost(@Path("id") int id);

    @GET("posts")
    Call<List<Post>> getAllPosts();

    @POST("posts")
    Call<Post> createPost(@Body Post post);
}
```

### 4.3 创建数据模型

```java
public class Post {
    private int userId;
    private int id;
    private String title;
    private String body;

    // 构造函数、getter和setter方法
    public Post(int userId, String title, String body) {
        this.userId = userId;
        this.title = title;
        this.body = body;
    }

    public int getUserId() {
        return userId;
    }

    public void setUserId(int userId) {
        this.userId = userId;
    }

    public int getId() {
        return id;
    }

    public void setId(int id) {
        this.id = id;
    }

    public String getTitle() {
        return title;
    }

    public void setTitle(String title) {
        this.title = title;
    }

    public String getBody() {
        return body;
    }

    public void setBody(String body) {
        this.body = body;
    }

    @Override
    public String toString() {
        return "Post{" +
                "userId=" + userId +
                ", id=" + id +
                ", title='" + title + '\'' +
                ", body='" + body + '\'' +
                '}';
    }
}
```

### 4.4 配置Retrofit实例

```java
public class RetrofitClient {
    private static final String BASE_URL = "https://jsonplaceholder.typicode.com/";
    private static Retrofit retrofit = null;

    public static Retrofit getClient() {
        if (retrofit == null) {
            OkHttpClient.Builder httpClient = new OkHttpClient.Builder();
            
            // 添加日志拦截器
            HttpLoggingInterceptor logging = new HttpLoggingInterceptor();
            logging.setLevel(HttpLoggingInterceptor.Level.BODY);
            httpClient.addInterceptor(logging);

            retrofit = new Retrofit.Builder()
                    .baseUrl(BASE_URL)
                    .addConverterFactory(GsonConverterFactory.create())
                    .client(httpClient.build())
                    .build();
        }
        return retrofit;
    }
}
```

### 4.5 使用Retrofit进行网络请求

```java
public class RetrofitDemoActivity extends AppCompatActivity {

    private ApiService apiService;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_retrofit_demo);

        apiService = RetrofitClient.getClient().create(ApiService.class);

        Button btnFetchPost = findViewById(R.id.btnFetchPost);
        btnFetchPost.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                fetchPost();
            }
        });

        Button btnCreatePost = findViewById(R.id.btnCreatePost);
        btnCreatePost.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                createPost();
            }
        });
    }

    private void fetchPost() {
        Call<Post> call = apiService.getPost(1);
        call.enqueue(new Callback<Post>() {
            @Override
            public void onResponse(Call<Post> call, Response<Post> response) {
                if (response.isSuccessful()) {
                    Post post = response.body();
                    Toast.makeText(RetrofitDemoActivity.this, 
                            "获取成功: " + post.getTitle(), 
                            Toast.LENGTH_SHORT).show();
                    Log.d("RetrofitDemo", post.toString());
                } else {
                    Toast.makeText(RetrofitDemoActivity.this, 
                            "错误码: " + response.code(), 
                            Toast.LENGTH_SHORT).show();
                }
            }

            @Override
            public void onFailure(Call<Post> call, Throwable t) {
                Toast.makeText(RetrofitDemoActivity.this, 
                        "请求失败: " + t.getMessage(), 
                        Toast.LENGTH_SHORT).show();
                Log.e("RetrofitDemo", "Error: " + t.getMessage());
            }
        });
    }

    private void createPost() {
        Post newPost = new Post(1, "新文章标题", "这是文章内容");
        Call<Post> call = apiService.createPost(newPost);
        call.enqueue(new Callback<Post>() {
            @Override
            public void onResponse(Call<Post> call, Response<Post> response) {
                if (response.isSuccessful()) {
                    Post createdPost = response.body();
                    Toast.makeText(RetrofitDemoActivity.this, 
                            "创建成功: ID = " + createdPost.getId(), 
                            Toast.LENGTH_SHORT).show();
                    Log.d("RetrofitDemo", "Created: " + createdPost.toString());
                } else {
                    Toast.makeText(RetrofitDemoActivity.this, 
                            "错误码: " + response.code(), 
                            Toast.LENGTH_SHORT).show();
                }
            }

            @Override
            public void onFailure(Call<Post> call, Throwable t) {
                Toast.makeText(RetrofitDemoActivity.this, 
                        "请求失败: " + t.getMessage(), 
                        Toast.LENGTH_SHORT).show();
                Log.e("RetrofitDemo", "Error: " + t.getMessage());
            }
        });
    }
}
```

## 5. 图片加载库 - Glide

Glide是Android中常用的图片加载库，可以高效地加载网络图片。

### 5.1 添加依赖

```gradle
dependencies {
    implementation 'com.github.bumptech.glide:glide:4.12.0'
    annotationProcessor 'com.github.bumptech.glide:compiler:4.12.0'
}
```

### 5.2 基本使用

```java
ImageView imageView = findViewById(R.id.imageView);

Glide.with(this)
    .load("https://example.com/image.jpg")
    .placeholder(R.drawable.placeholder)
    .error(R.drawable.error)
    .centerCrop()
    .into(imageView);
```

## 6. WebSocket通信

WebSocket提供了全双工通信通道，适合需要实时数据交换的应用场景。

### 6.1 添加OkHttp WebSocket依赖

```gradle
dependencies {
    implementation 'com.squareup.okhttp3:okhttp:4.9.0'
}
```

### 6.2 实现WebSocket客户端

```java
public class WebSocketActivity extends AppCompatActivity {

    private OkHttpClient client;
    private WebSocket webSocket;
    private TextView tvMessages;
    private EditText etMessage;
    private Button btnSend;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_websocket);

        tvMessages = findViewById(R.id.tvMessages);
        etMessage = findViewById(R.id.etMessage);
        btnSend = findViewById(R.id.btnSend);

        client = new OkHttpClient();
        connectWebSocket();

        btnSend.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                String message = etMessage.getText().toString().trim();
                if (!message.isEmpty()) {
                    webSocket.send(message);
                    etMessage.setText("");
                }
            }
        });
    }

    private void connectWebSocket() {
        Request request = new Request.Builder()
                .url("wss://echo.websocket.org")
                .build();

        webSocket = client.newWebSocket(request, new WebSocketListener() {
            @Override
            public void onOpen(WebSocket webSocket, Response response) {
                runOnUiThread(new Runnable() {
                    @Override
                    public void run() {
                        tvMessages.append("连接已建立\n");
                    }
                });
            }

            @Override
            public void onMessage(WebSocket webSocket, String text) {
                runOnUiThread(new Runnable() {
                    @Override
                    public void run() {
                        tvMessages.append("收到: " + text + "\n");
                    }
                });
            }

            @Override
            public void onClosing(WebSocket webSocket, int code, String reason) {
                webSocket.close(1000, null);
                runOnUiThread(new Runnable() {
                    @Override
                    public void run() {
                        tvMessages.append("连接关闭中: " + reason + "\n");
                    }
                });
            }

            @Override
            public void onFailure(WebSocket webSocket, Throwable t, Response response) {
                runOnUiThread(new Runnable() {
                    @Override
                    public void run() {
                        tvMessages.append("错误: " + t.getMessage() + "\n");
                    }
                });
            }
        });
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        // 关闭WebSocket连接
        if (webSocket != null) {
            webSocket.close(1000, "Activity销毁");
        }
    }
}

## 7. 总结

本文介绍了Android网络编程的核心内容：

1. 基本的HTTP请求（HttpURLConnection）
2. Retrofit库的使用
3. Glide图片加载
4. WebSocket实时通信

通过掌握这些技术，你可以在应用中实现各种网络功能，如数据获取、图片加载、实时通信等。在实际开发中，建议使用成熟的网络库（如Retrofit、OkHttp）来简化网络操作，提高开发效率。

## 8. 练习任务

1. 创建一个简单的新闻应用，使用Retrofit从API获取新闻数据
2. 实现一个图片浏览器，使用Glide加载网络图片
3. 开发一个简单的聊天应用，使用WebSocket实现实时通信
4. 为应用添加离线缓存功能，在无网络时仍能显示内容