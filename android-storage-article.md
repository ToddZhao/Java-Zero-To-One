# Day 78: Java安卓开发 - 数据存储和访问

## 引言

在Android应用开发中，数据存储和访问是非常重要的一环。本文将详细介绍Android中常见的数据存储方式，并通过实际案例来展示如何实现各种存储方案。我们将覆盖以下主要内容：

1. SharedPreferences存储
2. 文件存储
3. SQLite数据库存储
4. ContentProvider访问

## 1. SharedPreferences存储

SharedPreferences是Android中最简单的数据存储方式，适合存储少量的键值对数据，比如用户设置、登录状态等。

### 1.1 基本使用

```java
// 存储数据
SharedPreferences sharedPreferences = getSharedPreferences("app_settings", Context.MODE_PRIVATE);
SharedPreferences.Editor editor = sharedPreferences.edit();
editor.putString("username", "张三");
editor.putInt("age", 25);
editor.putBoolean("isLogin", true);
editor.apply(); // 或者使用commit()

// 读取数据
String username = sharedPreferences.getString("username", "默认用户");
int age = sharedPreferences.getInt("age", 0);
boolean isLogin = sharedPreferences.getBoolean("isLogin", false);
```

### 1.2 实际案例：用户设置管理器

```java
public class SettingsManager {
    private static final String PREF_NAME = "app_settings";
    private SharedPreferences sharedPreferences;

    public SettingsManager(Context context) {
        sharedPreferences = context.getSharedPreferences(PREF_NAME, Context.MODE_PRIVATE);
    }

    public void saveThemeMode(boolean isDarkMode) {
        sharedPreferences.edit().putBoolean("dark_mode", isDarkMode).apply();
    }

    public boolean isDarkMode() {
        return sharedPreferences.getBoolean("dark_mode", false);
    }

    public void saveNotificationSettings(boolean enabled) {
        sharedPreferences.edit().putBoolean("notifications_enabled", enabled).apply();
    }

    public boolean areNotificationsEnabled() {
        return sharedPreferences.getBoolean("notifications_enabled", true);
    }
}
```

## 2. 文件存储

Android提供了内部存储和外部存储两种文件存储方式。

### 2.1 内部存储

内部存储位于应用私有目录下，其他应用无法访问。

```java
public class FileHelper {
    public static void saveToInternalStorage(Context context, String filename, String content) {
        try {
            FileOutputStream fos = context.openFileOutput(filename, Context.MODE_PRIVATE);
            fos.write(content.getBytes());
            fos.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    public static String readFromInternalStorage(Context context, String filename) {
        StringBuilder content = new StringBuilder();
        try {
            FileInputStream fis = context.openFileInput(filename);
            BufferedReader reader = new BufferedReader(new InputStreamReader(fis));
            String line;
            while ((line = reader.readLine()) != null) {
                content.append(line).append("\n");
            }
            reader.close();
            fis.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
        return content.toString();
    }
}
```

### 2.2 外部存储

外部存储需要申请权限，但可以存储更大的文件。

```java
public class ExternalStorageHelper {
    public static void saveToExternalStorage(String filename, String content) {
        // 检查外部存储是否可用
        if (!isExternalStorageWritable()) {
            return;
        }

        File file = new File(Environment.getExternalStorageDirectory(), filename);
        try {
            FileOutputStream fos = new FileOutputStream(file);
            fos.write(content.getBytes());
            fos.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    public static boolean isExternalStorageWritable() {
        String state = Environment.getExternalStorageState();
        return Environment.MEDIA_MOUNTED.equals(state);
    }
}
```

## 3. SQLite数据库存储

SQLite是Android中内置的轻量级关系型数据库，适合存储结构化数据。

### 3.1 数据库帮助类

```java
public class DatabaseHelper extends SQLiteOpenHelper {
    private static final String DATABASE_NAME = "MyApp.db";
    private static final int DATABASE_VERSION = 1;

    public DatabaseHelper(Context context) {
        super(context, DATABASE_NAME, null, DATABASE_VERSION);
    }

    @Override
    public void onCreate(SQLiteDatabase db) {
        // 创建用户表
        String CREATE_USER_TABLE = "CREATE TABLE users (" +
                "id INTEGER PRIMARY KEY AUTOINCREMENT," +
                "username TEXT," +
                "email TEXT," +
                "created_at DATETIME DEFAULT CURRENT_TIMESTAMP)";
        db.execSQL(CREATE_USER_TABLE);
    }

    @Override
    public void onUpgrade(SQLiteDatabase db, int oldVersion, int newVersion) {
        db.execSQL("DROP TABLE IF EXISTS users");
        onCreate(db);
    }
}
```

### 3.2 数据操作示例

```java
public class UserDao {
    private SQLiteDatabase database;
    private DatabaseHelper dbHelper;

    public UserDao(Context context) {
        dbHelper = new DatabaseHelper(context);
    }

    public void open() {
        database = dbHelper.getWritableDatabase();
    }

    public void close() {
        dbHelper.close();
    }

    public long insertUser(String username, String email) {
        ContentValues values = new ContentValues();
        values.put("username", username);
        values.put("email", email);
        return database.insert("users", null, values);
    }

    public List<User> getAllUsers() {
        List<User> users = new ArrayList<>();
        Cursor cursor = database.query("users", null, null, null, null, null, null);
        
        if (cursor.moveToFirst()) {
            do {
                User user = new User();
                user.setId(cursor.getLong(cursor.getColumnIndex("id")));
                user.setUsername(cursor.getString(cursor.getColumnIndex("username")));
                user.setEmail(cursor.getString(cursor.getColumnIndex("email")));
                users.add(user);
            } while (cursor.moveToNext());
        }
        cursor.close();
        return users;
    }
}
```

## 4. ContentProvider

ContentProvider用于在不同应用间共享数据，是Android中实现跨进程数据访问的标准方式。

### 4.1 创建ContentProvider

```java
public class UserProvider extends ContentProvider {
    private DatabaseHelper dbHelper;
    private static final String AUTHORITY = "com.example.app.provider";
    public static final Uri CONTENT_URI = Uri.parse("content://" + AUTHORITY + "/users");

    @Override
    public boolean onCreate() {
        dbHelper = new DatabaseHelper(getContext());
        return true;
    }

    @Override
    public Cursor query(Uri uri, String[] projection, String selection,
                       String[] selectionArgs, String sortOrder) {
        SQLiteDatabase db = dbHelper.getReadableDatabase();
        return db.query("users", projection, selection, selectionArgs,
                null, null, sortOrder);
    }

    @Override
    public Uri insert(Uri uri, ContentValues values) {
        SQLiteDatabase db = dbHelper.getWritableDatabase();
        long id = db.insert("users", null, values);
        getContext().getContentResolver().notifyChange(uri, null);
        return ContentUris.withAppendedId(uri, id);
    }

    // 实现其他必要的方法...
}
```

### 4.2 访问ContentProvider

```java
public class ContentProviderDemo {
    public static void accessContentProvider(Context context) {
        // 查询数据
        Cursor cursor = context.getContentResolver().query(
            UserProvider.CONTENT_URI,
            null,
            null,
            null,
            null
        );

        if (cursor != null && cursor.moveToFirst()) {
            do {
                String username = cursor.getString(cursor.getColumnIndex("username"));
                String email = cursor.getString(cursor.getColumnIndex("email"));
                Log.d("ContentProviderDemo", "User: " + username + ", Email: " + email);
            } while (cursor.moveToNext());
            cursor.close();
        }

        // 插入数据
        ContentValues values = new ContentValues();
        values.put("username", "新用户");
        values.put("email", "new@example.com");
        Uri newUri = context.getContentResolver().insert(UserProvider.CONTENT_URI, values);
    }
}
```

## 总结

Android提供了多种数据存储方式，每种方式都有其适用场景：

- SharedPreferences适合存储简单的键值对数据
- 文件存储适合存储大量的原始数据
- SQLite数据库适合存储结构化数据
- ContentProvider适合需要跨应用共享的数据

在实际开发中，应根据具体需求选择合适的存储方式，同时注意数据的安全性和性能问题。建议在处理大量数据时使用异步操作，避免阻塞主线程。

## 参考资料

1. Android开发者文档
2. SQLite官方文档
3. Android存储最佳实践指南
