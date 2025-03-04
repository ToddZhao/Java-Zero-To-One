# Day 80: Java安卓开发 - 高级UI开发

## 1. 引言

在前面的教程中，我们已经学习了Android的基础知识和一些核心功能。本文将深入探讨Android的高级UI开发技术，包括Fragment、ViewPager、RecyclerView等组件的使用，帮助你构建更复杂、更专业的用户界面。

## 2. Fragment

Fragment是一种可以嵌入在Activity中的UI片段，它有自己的生命周期，可以让界面设计更加模块化和灵活。

### 2.1 Fragment基础

#### 创建Fragment

```java
public class HomeFragment extends Fragment {

    public HomeFragment() {
        // 必须的空构造函数
    }

    @Override
    public View onCreateView(LayoutInflater inflater, ViewGroup container,
                             Bundle savedInstanceState) {
        // 加载Fragment的布局
        return inflater.inflate(R.layout.fragment_home, container, false);
    }

    @Override
    public void onViewCreated(View view, Bundle savedInstanceState) {
        super.onViewCreated(view, savedInstanceState);
        
        // 初始化视图组件
        TextView tvTitle = view.findViewById(R.id.tvTitle);
        Button btnAction = view.findViewById(R.id.btnAction);
        
        btnAction.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                Toast.makeText(getContext(), "按钮被点击", Toast.LENGTH_SHORT).show();
            }
        });
    }
}
```

#### Fragment布局文件 (fragment_home.xml)

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:padding="16dp">

    <TextView
        android:id="@+id/tvTitle"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="首页"
        android:textSize="24sp"
        android:textStyle="bold" />

    <TextView
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_marginTop="8dp"
        android:text="这是首页Fragment的内容" />

    <Button
        android:id="@+id/btnAction"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_marginTop="16dp"
        android:text="点击我" />

</LinearLayout>
```

### 2.2 在Activity中使用Fragment

```java
public class FragmentDemoActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_fragment_demo);

        if (savedInstanceState == null) {
            // 添加Fragment到Activity
            getSupportFragmentManager().beginTransaction()
                    .add(R.id.fragment_container, new HomeFragment())
                    .commit();
        }

        // 设置底部导航切换Fragment
        BottomNavigationView bottomNav = findViewById(R.id.bottom_navigation);
        bottomNav.setOnNavigationItemSelectedListener(new BottomNavigationView.OnNavigationItemSelectedListener() {
            @Override
            public boolean onNavigationItemSelected(@NonNull MenuItem item) {
                Fragment selectedFragment = null;

                switch (item.getItemId()) {
                    case R.id.nav_home:
                        selectedFragment = new HomeFragment();
                        break;
                    case R.id.nav_dashboard:
                        selectedFragment = new DashboardFragment();
                        break;
                    case R.id.nav_notifications:
                        selectedFragment = new NotificationsFragment();
                        break;
                }

                if (selectedFragment != null) {
                    getSupportFragmentManager().beginTransaction()
                            .replace(R.id.fragment_container, selectedFragment)
                            .commit();
                }

                return true;
            }
        });
    }
}
```

#### Activity布局文件 (activity_fragment_demo.xml)

```xml
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <FrameLayout
        android:id="@+id/fragment_container"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:layout_above="@id/bottom_navigation" />

    <com.google.android.material.bottomnavigation.BottomNavigationView
        android:id="@+id/bottom_navigation"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_alignParentBottom="true"
        app:menu="@menu/bottom_navigation_menu" />

</RelativeLayout>
```

### 2.3 Fragment间通信

#### 使用接口

```java
public class ItemFragment extends Fragment {

    private OnItemSelectedListener listener;

    // 定义接口
    public interface OnItemSelectedListener {
        void onItemSelected(String item);
    }

    @Override
    public void onAttach(Context context) {
        super.onAttach(context);
        if (context instanceof OnItemSelectedListener) {
            listener = (OnItemSelectedListener) context;
        } else {
            throw new RuntimeException(context.toString()
                    + " 必须实现OnItemSelectedListener接口");
        }
    }

    // 在Fragment中调用接口方法
    private void selectItem(String item) {
        if (listener != null) {
            listener.onItemSelected(item);
        }
    }
}
```

#### 在Activity中实现接口

```java
public class CommunicationActivity extends AppCompatActivity 
        implements ItemFragment.OnItemSelectedListener {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_communication);

        if (savedInstanceState == null) {
            getSupportFragmentManager().beginTransaction()
                    .add(R.id.fragment_container, new ItemFragment())
                    .commit();
        }
    }

    @Override
    public void onItemSelected(String item) {
        // 处理从Fragment传来的数据
        DetailFragment detailFragment = DetailFragment.newInstance(item);
        
        getSupportFragmentManager().beginTransaction()
                .replace(R.id.detail_container, detailFragment)
                .addToBackStack(null)
                .commit();
    }
}
```

## 3. ViewPager2

ViewPager2是Android提供的滑动页面组件，常用于实现引导页、图片轮播等功能。

### 3.1 基本使用

```java
public class ViewPagerDemoActivity extends AppCompatActivity {

    private ViewPager2 viewPager;
    private TabLayout tabLayout;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_viewpager_demo);

        viewPager = findViewById(R.id.viewPager);
        tabLayout = findViewById(R.id.tabLayout);

        // 创建适配器
        ViewPagerAdapter adapter = new ViewPagerAdapter(this);
        viewPager.setAdapter(adapter);

        // 将TabLayout与ViewPager2关联
        new TabLayoutMediator(tabLayout, viewPager,
                (tab, position) -> tab.setText("页面 " + (position + 1)))
                .attach();
    }

    // ViewPager2适配器
    private class ViewPagerAdapter extends FragmentStateAdapter {

        public ViewPagerAdapter(FragmentActivity fragmentActivity) {
            super(fragmentActivity);
        }

        @NonNull
        @Override
        public Fragment createFragment(int position) {
            // 根据位置返回不同的Fragment
            switch (position) {
                case 0:
                    return new HomeFragment();
                case 1:
                    return new DashboardFragment();
                case 2:
                    return new NotificationsFragment();
                default:
                    return new HomeFragment();
            }
        }

        @Override
        public int getItemCount() {
            return 3; // 页面总数
        }
    }
}
```

### 3.2 布局文件 (activity_viewpager_demo.xml)

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical">

    <com.google.android.material.tabs.TabLayout
        android:id="@+id/tabLayout"
        android:layout_width="match_parent"
        android:layout_height="wrap_content" />

    <androidx.viewpager2.widget.ViewPager2
        android:id="@+id/viewPager"
        android:layout_width="match_parent"
        android:layout_height="match_parent" />

</LinearLayout>
```

## 4. RecyclerView

RecyclerView是一个强大的视图组件，用于高效地显示大量数据。

### 4.1 基本使用

#### 数据模型

```java
public class User {
    private String name;
    private String email;
    private int avatarResId;

    public User(String name, String email, int avatarResId) {
        this.name = name;
        this.email = email;
        this.avatarResId = avatarResId;
    }

    public String getName() {
        return name;
    }

    public String getEmail() {
        return email;
    }

    public int getAvatarResId() {
        return avatarResId;
    }
}
```

#### 适配器

```java
public class UserAdapter extends RecyclerView.Adapter<UserAdapter.UserViewHolder> {

    private List<User> userList;
    private OnItemClickListener listener;

    // 定义点击监听接口
    public interface OnItemClickListener {
        void onItemClick(User user, int position);
    }

    public UserAdapter(List<User> userList) {
        this.userList = userList;
    }

    public void setOnItemClickListener(OnItemClickListener listener) {
        this.listener = listener;
    }

    @NonNull
    @Override
    public UserViewHolder onCreateViewHolder(@NonNull ViewGroup parent, int viewType) {
        View view = LayoutInflater.from(parent.getContext())
                .inflate(R.layout.item_user, parent, false);
        return new UserViewHolder(view);
    }

    @Override
    public void onBindViewHolder(@NonNull UserViewHolder holder, int position) {
        User user = userList.get(position);
        holder.tvName.setText(user.getName());
        holder.tvEmail.setText(user.getEmail());
        holder.ivAvatar.setImageResource(user.getAvatarResId());

        holder.itemView.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                if (listener != null) {
                    int pos = holder.getAdapterPosition();
                    if (pos != RecyclerView.NO_POSITION) {
                        listener.onItemClick(userList.get(pos), pos);
                    }
                }
            }
        });
    }

    @Override
    public int getItemCount() {
        return userList.size();
    }

    public static class UserViewHolder extends RecyclerView.ViewHolder {
        ImageView ivAvatar;
        TextView tvName;
        TextView tvEmail;

        public UserViewHolder(@NonNull View itemView) {
            super(itemView);
            ivAvatar = itemView.findViewById(R.id.ivAvatar);
            tvName = itemView.findViewById(R.id.tvName);
            tvEmail = itemView.findViewById(R.id.tvEmail);
        }
    }
}
```

#### 列表项布局 (item_user.xml)

```xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.cardview.widget.CardView xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:layout_margin="8dp"
    app:cardCornerRadius="8dp"
    app:cardElevation="4dp">

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="horizontal"
        android:padding="16dp">

        <ImageView
            android:id="@+id/ivAvatar"
            android:layout_width="50dp"
            android:layout_height="50dp"
            android:src="@drawable/avatar_placeholder" />

        <LinearLayout
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:layout_marginStart="16dp"
            android:orientation="vertical">

            <TextView
                android:id="@+id/tvName"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:text="用户名"
                android:textSize="16sp"
                android:textStyle="bold" />

            <TextView
                android:id="@+id/tvEmail"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:layout_marginTop="4dp"
                android:text="邮箱地址"
                android:textSize="14sp" />
        </LinearLayout>
    </LinearLayout>
</androidx.cardview.widget.CardView>
```

#### 在Activity中使用RecyclerView

```java
public class RecyclerViewDemoActivity extends AppCompatActivity {

    private RecyclerView recyclerView;
    private UserAdapter adapter;
    private List<User> userList;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_recyclerview_demo);

        recyclerView = findViewById(R.id.recyclerView);
        recyclerView.setLayoutManager(new LinearLayoutManager(this));

        // 添加分割线
        recyclerView.addItemDecoration(new DividerItemDecoration(this, DividerItemDecoration.VERTICAL));

        // 准备数据
        prepareUserData();

        // 设置适配器
        adapter = new UserAdapter(userList);
        recyclerView.setAdapter(adapter);

        // 设置点击事件
        adapter.setOnItemClickListener(new UserAdapter.OnItemClickListener() {
            @Override
            public void onItemClick(User user, int position) {
                Toast.makeText(RecyclerViewDemoActivity.this,
                        "选择了: " + user.getName(),
                        Toast.LENGTH_SHORT).show();
            }
        });
    }

    private void prepareUserData() {
        userList = new ArrayList<>();
        userList.add(new User("张三", "zhangsan@example.com", R.drawable.avatar1));
        userList.add(new User("李四", "lisi@example.com", R.drawable.avatar2));
        userList.add(new User("王五", "wangwu@example.com", R.drawable.avatar3));
        userList.add(new User("赵六", "zhaoliu@example.com", R.drawable.avatar4));
        // 添加更多用户数据...
    }
}
```

#### 布局文件 (activity_recyclerview_demo.xml)

```xml
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <androidx.recyclerview.widget.RecyclerView
        android:id="@+id/recyclerView"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:scrollbars="vertical" />

</RelativeLayout>
```

## 5. 高级UI效果

### 5.1 卡片滑动效果 (ItemTouchHelper)

```java
public class SwipeToDeleteCallback extends ItemTouchHelper.SimpleCallback {

    private UserAdapter adapter;

    public SwipeToDeleteCallback(UserAdapter adapter) {
        super(0, ItemTouchHelper.LEFT | ItemTouchHelper.RIGHT);
        this.adapter = adapter;
    }

    @Override
    public boolean onMove(@NonNull RecyclerView recyclerView, @NonNull RecyclerView.ViewHolder viewHolder, @NonNull RecyclerView.ViewHolder target) {
        return false;
    }

    @Override
    public void onSwiped(@NonNull RecyclerView.ViewHolder viewHolder, int direction) {
        int position = viewHolder.getAdapterPosition();
        adapter.deleteItem(position);
    }
}
```

在Activity中添加滑动删除功能：

```java
// 添加滑动删除功能
ItemTouchHelper itemTouchHelper = new ItemTouchHelper(new SwipeToDeleteCallback(adapter));
itemTouchHelper.attachToRecyclerView(recyclerView);
```

### 5.2 动画效果

```java
// 为RecyclerView添加动画
recyclerView.setItemAnimator(new DefaultItemAnimator());

// 自定义进入动画
private void runLayoutAnimation(RecyclerView recyclerView) {
    Context context = recyclerView.getContext();
    LayoutAnimationController animation = AnimationUtils.loadLayoutAnimation(
            context, R.anim.layout_animation_fall_down);
    recyclerView.setLayoutAnimation(animation);
    recyclerView.scheduleLayoutAnimation();
}
```

## 6. 总结

本文介绍了Android高级UI开发的核心组件和技术：

1. **Fragment**：模块化UI组件，有自己的生命周期，可以灵活组合
2. **ViewPager2**：滑动页面组件，用于实现引导页、图片轮播等
3. **RecyclerView**：高效显示大量数据的列表组件
4. **高级UI效果**：滑动删除、动画等交互效果

通过掌握这些高级UI组件，你可以构建更复杂、更专业的用户界面，提升应用的用户体验。在实际开发中，合理使用这些组件可以使代码更加模块化、可维护，同时提供更流畅的用户交互。

## 7. 练习任务

1. 创建一个新闻应用，使用Fragment实现新闻列表和详情页
2. 实现一个图片浏览器，使用ViewPager2实现图片滑动浏览
3. 开发一个联系人应用，使用RecyclerView显示联系人列表，并实现滑动删除功能
4. 为应用添加动画效果，提升用户体验