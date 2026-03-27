---
title: "[Android] AIDL 이란?"
excerpt: "AIDL 란?"

categories:
  - Android
tags:
  - [tag1, tag2]

permalink: /Android/AIDL/

toc: true
toc_sticky: true

date: 2025-10-23
last_modified_at: 2026-03-27
---

# 전체구조
```
AppA (Provider)
 ├─ IRemoteService.aidl
 ├─ IRemoteCallback.aidl
 ├─ RemoteService.java (Service 구현)
 └─ AndroidManifest.xml (서비스 선언)

AppB (Client)
 ├─ IRemoteService.aidl (같은 내용 복사)
 ├─ IRemoteCallback.aidl (같은 내용 복사)
 ├─ MainActivity.java (서비스 연결 및 콜백 등록)
```
## 📌 실행 흐름 요약

|순서|동작|설명|
|---|---|---|
|①|Client가 Provider의 Service 바인드|`bindService()` 호출|
|②|Service 연결 후 Callback 등록|`mService.registerCallback()`|
|③|Client → Service|`mService.sendMessage("Hello")`|
|④|Service가 모든 콜백에 알림|`callback.onMessageReceived()` 호출|
|⑤|Client UI 업데이트|`runOnUiThread()`로 TextView에 표시|

## 📌 핵심 포인트
- 콜백은 `RemoteCallbackList<T>`로 관리해야 다중 클라이언트 안전하게 처리 가능.
- AIDL 인터페이스는 양쪽에 **패키지, 파일명, 메서드 시그니처 모두 동일**해야 함.
- 양방향 통신이므로 `Stub` ↔ `Proxy` 구조를 자동 생성함.
- 실제로는 `Binder`를 통해 Android IPC가 수행됨.

## 1. AIDL 인터페이스 정의
- **주의:**  AIDL 파일은 양쪽 앱에 **동일한 패키지 경로와 내용**으로 존재
  (빌드 시 자동으로 Stub 클래스가 생성되어 IPC가 가능해짐)

### IRemoteCallback.aidl
```c
package com.example.remoteservice;

interface IRemoteCallback {
    void onMessageReceived(String message);
}
```

### IRemoteService.aidl
```c
package com.example.remoteservice;

import com.example.remoteservice.IRemoteCallback;

interface IRemoteService {
    void registerCallback(IRemoteCallback callback);
    void unregisterCallback(IRemoteCallback callback);
    void sendMessage(String message);
}
```

## 2. App A (Provider)
- 서비스 구현
### RemoteService.java

```c
package com.example.remoteservice;

import android.app.Service;
import android.content.Intent;
import android.os.IBinder;
import android.os.RemoteCallbackList;
import android.os.RemoteException;
import android.util.Log;

public class RemoteService extends Service {
    private static final String TAG = "RemoteService";

    private final RemoteCallbackList<IRemoteCallback> mCallbacks = new RemoteCallbackList<>();

    private final IRemoteService.Stub mBinder = new IRemoteService.Stub() {
        @Override
        public void registerCallback(IRemoteCallback callback) {
            if (callback != null) {
                mCallbacks.register(callback);
                Log.d(TAG, "Callback registered");
            }
        }

        @Override
        public void unregisterCallback(IRemoteCallback callback) {
            if (callback != null) {
                mCallbacks.unregister(callback);
                Log.d(TAG, "Callback unregistered");
            }
        }

        @Override
        public void sendMessage(String message) throws RemoteException {
            Log.d(TAG, "Received message from client: " + message);
            broadcastToClients("Echo from Service: " + message);
        }
    };

    private void broadcastToClients(String message) throws RemoteException {
        final int count = mCallbacks.beginBroadcast();
        for (int i = 0; i < count; i++) {
            try {
                mCallbacks.getBroadcastItem(i).onMessageReceived(message);
            } catch (Exception e) {
                Log.e(TAG, "Callback error", e);
            }
        }
        mCallbacks.finishBroadcast();
    }

    @Override
    public IBinder onBind(Intent intent) {
        Log.d(TAG, "Service bound");
        return mBinder;
    }
}
```

### AndroidManifest.xml
- `protectionLevel="signature"`로 지정하면 같은 서명 앱만 접근 가능.  
테스트 시엔 제거해도 무방 (`android:permission` 생략 가능).

```c
<service
    android:name=".RemoteService"
    android:exported="true"
    android:permission="com.example.remoteservice.BIND_SERVICE">
    <intent-filter>
        <action android:name="com.example.remoteservice.RemoteService" />
    </intent-filter>
</service>

<permission
    android:name="com.example.remoteservice.BIND_SERVICE"
    android:protectionLevel="signature" />
```

## 3. App B (Client)
- 서비스 연결 및 콜백 등록
### MainActivity.java

```c
package com.example.remoteclient;

import android.content.ComponentName;
import android.content.Context;
import android.content.Intent;
import android.content.ServiceConnection;
import android.os.Bundle;
import android.os.IBinder;
import android.os.RemoteException;
import android.util.Log;
import android.widget.Button;
import android.widget.EditText;
import android.widget.TextView;
import androidx.appcompat.app.AppCompatActivity;
import com.example.remoteservice.IRemoteCallback;
import com.example.remoteservice.IRemoteService;

public class MainActivity extends AppCompatActivity {
    private static final String TAG = "RemoteClient";

    private IRemoteService mService;
    private boolean mBound = false;
    private TextView textView;
    private EditText editText;

    private final IRemoteCallback mCallback = new IRemoteCallback.Stub() {
        @Override
        public void onMessageReceived(String message) {
            runOnUiThread(() -> textView.append("From Service: " + message + "\n"));
        }
    };

    private final ServiceConnection mConnection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            mService = IRemoteService.Stub.asInterface(service);
            try {
                mService.registerCallback(mCallback);
            } catch (RemoteException e) {
                Log.e(TAG, "registerCallback failed", e);
            }
            mBound = true;
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {
            try {
                if (mService != null) {
                    mService.unregisterCallback(mCallback);
                }
            } catch (RemoteException ignored) {}
            mBound = false;
            mService = null;
        }
    };

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        textView = findViewById(R.id.textView);
        editText = findViewById(R.id.editText);
        Button sendButton = findViewById(R.id.sendButton);

        sendButton.setOnClickListener(v -> {
            if (mBound && mService != null) {
                try {
                    mService.sendMessage(editText.getText().toString());
                } catch (RemoteException e) {
                    Log.e(TAG, "sendMessage failed", e);
                }
            }
        });

        bindToRemoteService();
    }

    private void bindToRemoteService() {
        Intent intent = new Intent("com.example.remoteservice.RemoteService");

        intent.setPackage("com.example.remoteservice"); // Provider 패키지명
        bindService(intent, mConnection, Context.BIND_AUTO_CREATE);
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        
        if (mBound) {
            try {
                mService.unregisterCallback(mCallback);
            } catch (RemoteException ignored) {}
            unbindService(mConnection);
        }
    }
}
```
