---
title: "[Android] MVVM 이란?"
excerpt: "MVVM 란?"

categories:
  - Android
tags:
  - [tag1, tag2]

permalink: /Android/MVVM/

toc: true
toc_sticky: true

date: 2025-11-25
last_modified_at: 2026-03-30
---

## 1. MVVM 개요
MVVM은 **Model-View-ViewModel**의 약자로, UI 관련 코드와 비즈니스 로직을 분리하는 디자인 패턴
- **Model**: 데이터를 관리하는 계층 (RoomDB, Repository)
- **View**: UI 계층 (Activity, Fragment)
- **ViewModel**: View와 Model 사이에서 데이터를 중개하며, UI 로직을 포함

**View가 직접 Model에 접근하지 않고**, ViewModel 통해 데이터를 가져오고 UI를 업데이트

## 2. MVVM + Room
- **ViewModel**은 UI 데이터를 관리하고, Activity/Fragment가 직접 DB와 통신하지 않도록 함  
-  **RoomDB**는 데이터를 로컬에서 저장하고 관리  
-  **LiveData**를 통한 UI가 자동 업데이트
-  **Repository**는 ViewModel과 RoomDB 사이에서 데이터를 중개

## 3. ViewModel과 RoomDB의 관계
RoomDB는 **로컬 데이터베이스**이고, ViewModel은 이를 가져와 UI에 제공하는 역할
데이터 흐름 \
1) **View**(Activity/Fragment)가 UI 이벤트 발생 시 ViewModel에 요청 \
2) **ViewModel**은 Repository를 통해 RoomDB에서 데이터를 가져옴 \
3) RoomDB는 **LiveData** 또는 **Flow** 형태로 데이터를 제공 \
4) ViewModel은 데이터를 가공하여 View에 전달 \
5) View는 ViewModel의 LiveData를 확인하여 UI 업데이트

## 4. MVVM + RoomDB 코드
### (1) Entity - 데이터 테이블 정의
```java
@Entity(tableName = "message_table")
public class Message {
    @PrimaryKey(autoGenerate = true)
    public int id;

    public String content;

    public Message(String content) {
        this.content = content;
    }
}
```

### (2) DAO - 데이터 접근 객체
```java
@Dao
public interface MessageDao {
    @Insert
    void insert(Message message);

    @Query("SELECT * FROM message_table ORDER BY id DESC")
    LiveData<List<Message>> getAllMessages();
}
```

### (3) RoomDB - Database 객체
```java
@Database(entities = {Message.class}, version = 1)
public abstract class AppDatabase extends RoomDatabase {
    private static volatile AppDatabase INSTANCE;

    public abstract MessageDao messageDao();

    public static AppDatabase getDatabase(Context context) {
        if (INSTANCE == null) {
            synchronized (AppDatabase.class) {
                if (INSTANCE == null) {
                    INSTANCE = Room.databaseBuilder(
                            context.getApplicationContext(),
                            AppDatabase.class, "app_database")
                            .build();
                }
            }
        }
        return INSTANCE;
    }
}
```

### (4) Repository - 데이터 관리 계층
```java
public class MessageRepository {
    private MessageDao messageDao;
    private LiveData<List<Message>> allMessages;

    public MessageRepository(Application application) {
        AppDatabase db = AppDatabase.getDatabase(application);
        messageDao = db.messageDao();
        allMessages = messageDao.getAllMessages();
    }

    public LiveData<List<Message>> getAllMessages() {
        return allMessages;
    }

    public void insert(Message message) {
        Executors.newSingleThreadExecutor().execute(() -> messageDao.insert(message));
    }
}
```

### (5) ViewModel - 데이터 처리 계층

```java
public class MessageViewModel extends AndroidViewModel {
    private MessageRepository repository;
    private LiveData<List<Message>> allMessages;

    public MessageViewModel(@NonNull Application application) {
        super(application);
        repository = new MessageRepository(application);
        allMessages = repository.getAllMessages();
    }

    public LiveData<List<Message>> getAllMessages() {
        return allMessages;
    }

    public void insert(Message message) {
        repository.insert(message);
    }
}
```

### (6) Activity - UI 연결
```java
public class MainActivity extends AppCompatActivity {
    private MessageViewModel messageViewModel;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        messageViewModel = new ViewModelProvider(this).get(MessageViewModel.class);

        messageViewModel.getAllMessages().observe(this, messages -> {
            // UI 업데이트 (예: RecyclerView)
            for (Message msg : messages) {
                Log.d("MainActivity", "Message: " + msg.content);
            }
        });

        findViewById(R.id.addMessageButton).setOnClickListener(v -> {
            messageViewModel.insert(new Message("Hello MVVM!"));
        });
    }
}
```
