# Day 77: Java安卓开发 - 数据存储

## 1. 引言

数据存储是Android应用开发中的重要组成部分。本文将详细介绍Android中的各种数据存储方式，包括SharedPreferences、文件存储和SQLite数据库。

## 2. SharedPreferences

SharedPreferences是Android提供的一种轻量级的数据存储方式，适合存储少量的键值对数据。

### 2.1 基本使用

```java
public class PreferencesActivity extends AppCompatActivity {
    private SharedPreferences sharedPreferences;
    private SharedPreferences.Editor editor;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_preferences);

        // 获取SharedPreferences实例
        sharedPreferences = getSharedPreferences("MyPrefs", MODE_PRIVATE);
        editor = sharedPreferences.edit();

        // 存储数据
        saveData();

        // 读取数据
        loadData();
    }

    private void saveData() {
        editor.putString("username", "张三");
        editor.putInt("age", 25);
        editor.putBoolean("isVIP", true);
        editor.apply(); // 或使用commit()
    }

    private void loadData() {
        String username = sharedPreferences.getString("username", "");
        int age = sharedPreferences.getInt("age", 0);
        boolean isVIP = sharedPreferences.getBoolean("isVIP", false);

        Log.d("PreferencesDemo", "用户名: " + username);
        Log.d("PreferencesDemo", "年龄: " + age);
        Log.d("PreferencesDemo", "是否VIP: " + isVIP);
    }
}
```

## 3. 文件存储

Android提供了内部存储和外部存储两种文件存储方式。

### 3.1 内部存储

```java
public class FileStorageActivity extends AppCompatActivity {
    private static final String FILE_NAME = "note.txt";

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_file_storage);

        // 写入文件
        writeFile();

        // 读取文件
        readFile();
    }

    private void writeFile() {
        String data = "这是要保存的数据";
        try {
            FileOutputStream fos = openFileOutput(FILE_NAME, MODE_PRIVATE);
            fos.write(data.getBytes());
            fos.close();
            Toast.makeText(this, "数据保存成功", Toast.LENGTH_SHORT).show();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    private void readFile() {
        try {
            FileInputStream fis = openFileInput(FILE_NAME);
            InputStreamReader isr = new InputStreamReader(fis);
            BufferedReader br = new BufferedReader(isr);
            StringBuilder sb = new StringBuilder();
            String line;
            while ((line = br.readLine()) != null) {
                sb.append(line);
            }
            Toast.makeText(this, "读取的数据: " + sb.toString(), Toast.LENGTH_SHORT).show();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

### 3.2 外部存储

```java
public class ExternalStorageActivity extends AppCompatActivity {
    private static final String FILE_NAME = "external_note.txt";

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_external_storage);

        // 检查权限
        checkPermission();
    }

    private void checkPermission() {
        if (ContextCompat.checkSelfPermission(this,
                Manifest.permission.WRITE_EXTERNAL_STORAGE)
                != PackageManager.PERMISSION_GRANTED) {
            ActivityCompat.requestPermissions(this,
                    new String[]{Manifest.permission.WRITE_EXTERNAL_STORAGE},
                    1);
        } else {
            writeExternalFile();
        }
    }

    private void writeExternalFile() {
        String data = "这是要保存到外部存储的数据";
        File file = new File(getExternalFilesDir(null), FILE_NAME);

        try {
            FileOutputStream fos = new FileOutputStream(file);
            fos.write(data.getBytes());
            fos.close();
            Toast.makeText(this, "数据保存到外部存储成功", Toast.LENGTH_SHORT).show();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

## 4. SQLite数据库

SQLite是Android中内置的轻量级关系型数据库。

### 4.1 创建数据库帮助类

```java
public class DatabaseHelper extends SQLiteOpenHelper {
    private static final String DATABASE_NAME = "UserDB";
    private static final int DATABASE_VERSION = 1;

    public DatabaseHelper(Context context) {
        super(context, DATABASE_NAME, null, DATABASE_VERSION);
    }

    @Override
    public void onCreate(SQLiteDatabase db) {
        String CREATE_TABLE = "CREATE TABLE users (" +
                "id INTEGER PRIMARY KEY AUTOINCREMENT, " +
                "name TEXT, " +
                "age INTEGER)";
        db.execSQL(CREATE_TABLE);
    }

    @Override
    public void onUpgrade(SQLiteDatabase db, int oldVersion, int newVersion) {
        db.execSQL("DROP TABLE IF EXISTS users");
        onCreate(db);
    }
}
```

### 4.2 数据库操作

```java
public class DatabaseActivity extends AppCompatActivity {
    private DatabaseHelper dbHelper;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_database);

        dbHelper = new DatabaseHelper(this);

        // 插入数据
        insertUser();

        // 查询数据
        queryUsers();
    }

    private void insertUser() {
        SQLiteDatabase db = dbHelper.getWritableDatabase();

        ContentValues values = new ContentValues();
        values.put("name", "李四");
        values.put("age", 30);

        long newRowId = db.insert("users", null, values);
        Toast.makeText(this, "插入数据ID: " + newRowId, Toast.LENGTH_SHORT).show();
    }

    private void queryUsers() {
        SQLiteDatabase db = dbHelper.getReadableDatabase();

        String[] projection = {"id", "name", "age"};

        Cursor cursor = db.query(
                "users",
                projection,
                null,
                null,
                null,
                null,
                null
        );

        while (cursor.moveToNext()) {
            int id = cursor.getInt(cursor.getColumnIndexOrThrow("id"));
            String name = cursor.getString(cursor.getColumnIndexOrThrow("name"));
            int age = cursor.getInt(cursor.getColumnIndexOrThrow("age"));

            Log.d("DatabaseDemo", "ID: " + id + ", 姓名: " + name + ", 年龄: " + age);
        }
        cursor.close();
    }

    @Override
    protected void onDestroy() {
        dbHelper.close();
        super.onDestroy();
    }
}
```

## 5. 最佳实践

1. **选择合适的存储方式**
   - SharedPreferences：适合存储简单的键值对数据
   - 文件存储：适合存储大量的原始数据
   - SQLite数据库：适合存储结构化数据

2. **数据安全性考虑**
   - 敏感数据加密存储
   - 使用内部存储保护私密数据
   - 定期备份重要数据

3. **性能优化**
   - 批量操作数据库
   - 适时关闭数据库连接
   - 避免在主线程进行耗时操作

## 6. 练习任务

1. 创建一个简单的记事本应用，使用SQLite存储笔记内容
2. 实现用户设置功能，使用SharedPreferences保存用户偏好
3. 开发文件下载功能，将下载的文件保存到外部存储
4. 实现数据导入导出功能，支持数据备份和恢复

## 7. 总结

本文介绍了Android中三种主要的数据存储方式：
- SharedPreferences
- 文件存储（内部和外部）
- SQLite数据库

通过合理使用这些存储方式，我们可以有效地管理应用数据，提供更好的用户体验。在实际开发中，需要根据具体需求选择合适的存储方式，并注意数据安全和性能优化。