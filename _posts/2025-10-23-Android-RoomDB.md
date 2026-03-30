---
title: "[Android] RoomDB 이란?"
excerpt: "RoomDB 란?"

categories:
  - Android
tags:
  - [tag1, tag2]

permalink: /Android/RoomDB/

toc: true
toc_sticky: true

date: 2025-03-25
last_modified_at: 2026-03-30
---

`Room Database`는 SQLite를 더 쉽게 사용할 수 있도록 도와주는 **Jetpack 라이브러리**
- **객체 지향적인 API 제공** → SQL 문을 직접 작성하지 않고 `DAO`로 데이터를 관리 
- **SQLite보다 간편하고 안전** → `@Entity`, `@Dao` 등의 애너테이션으로 데이터 관리 가능
- **LiveData, Flow 지원** → 데이터 변경을 자동 감지하여 UI 업데이트 가능

Room 라이브러리 아키텍처 다이어그램
![[Pasted image 20250325115813.png]]
✅ **RoomDB는 SQLite보다 사용이 간편하고 안전함**  
✅ **DAO를 통해 SQL을 직접 작성하지 않고도 데이터 조작 가능**  
✅ **LiveData와 함께 사용하면 UI가 자동으로 업데이트됨**  
✅ **백그라운드 스레드에서 DB 작업을 수행해야 함**
![[Pasted image 20250325115938.png]]


## **RoomDB 기본 구조**
### **(1) Entity (데이터 테이블 정의)**
```java
@Entity(tableName = "users")
public class User {
    @PrimaryKey(autoGenerate = true)
    public int id;

    @ColumnInfo(name = "name")
    public String name;

    public User(String name) {
        this.name = name;
    }
}
```

- `@Entity` → SQLite 테이블을 나타냄
- `@PrimaryKey(autoGenerate = true)` → 자동 증가하는 기본 키
- `@ColumnInfo(name = "name")` → 컬럼 이름 설정 가능


### **(2) DAO (데이터 접근 객체)**
``` java
@Dao
public interface UserDao {
    @Insert
    void insert(User user);

    @Query("SELECT * FROM users")
    LiveData<List<User>> getAllUsers();
}
```
- `@Insert` → 데이터를 삽입하는 메서드
- `@Query` → SQL 쿼리를 직접 작성하여 데이터 조회 가능
- **LiveData 사용** → 데이터 변경 시 UI 자동 업데이트


### **(3) Database (Room Database 정의)**
```java
@Database(entities = {User.class}, version = 1)
public abstract class AppDatabase extends RoomDatabase {
    private static volatile AppDatabase INSTANCE;

    public abstract UserDao userDao();

    public static AppDatabase getDatabase(final Context context) {
        if (INSTANCE == null) {
            synchronized (AppDatabase.class) {
                if (INSTANCE == null) {
                    INSTANCE = Room.databaseBuilder(
                            context.getApplicationContext(),
                            AppDatabase.class, "app_database"
                    ).build();
                }
            }
        }
        return INSTANCE;
    }
}
```

- `@Database(entities = {User.class}, version = 1)` → 사용할 엔터티(User)와 데이터베이스 버전 지정
- `singleton 패턴` 사용 → **데이터베이스 인스턴스가 하나만 생성되도록 보장**


### **(4) ViewModel (UI와 RoomDB 연결)**
```java
public class UserViewModel extends AndroidViewModel {
    private final UserDao userDao;
    private final LiveData<List<User>> allUsers;

    public UserViewModel(@NonNull Application application) {
        super(application);
        AppDatabase db = AppDatabase.getDatabase(application);
        userDao = db.userDao();
        allUsers = userDao.getAllUsers();
    }

    public LiveData<List<User>> getAllUsers() {
        return allUsers;
    }

    public void insert(User user) {
        Executors.newSingleThreadExecutor().execute(() -> userDao.insert(user));
    }
}
```
- `AndroidViewModel` 사용 → RoomDB와 UI 간 데이터 관리
- `LiveData<List<User>>`를 사용하여 **UI가 자동으로 업데이트**됨
- `Executors.newSingleThreadExecutor().execute()` → **RoomDB 작업은 백그라운드 스레드에서 실행해야 함**


### **(5) Activity에서 RoomDB 사용**
```java
public class MainActivity extends AppCompatActivity {
    private UserViewModel userViewModel;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        userViewModel = new ViewModelProvider(this).get(UserViewModel.class);

        userViewModel.getAllUsers().observe(this, users -> {
            for (User user : users) {
                Log.d("MainActivity", "User: " + user.name);
            }
        });

        findViewById(R.id.addUserButton).setOnClickListener(v -> {
            userViewModel.insert(new User("John Doe"));
        });
    }
}

- **ViewModel을 통해 데이터 가져오기** → `getAllUsers().observe()`를 사용하여 UI 자동 업데이트
- **버튼 클릭 시 데이터 삽입** → `userViewModel.insert(new User("John Doe"))`

