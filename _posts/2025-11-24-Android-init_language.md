---
title: "[Android] init language 이란?"
excerpt: "init_language 란?"

categories:
  - Android
tags:
  - [tag1, tag2]

permalink: /Android/init_language/

toc: true
toc_sticky: true

date: 2025-05-28
last_modified_at: 2026-03-31
---

## 기본 구조
`init.rc` 파일은 **명령(command)** 와 **섹션(section)** 으로 구성

### 1. **서비스 섹션** (`service`)
서비스를 정의하고 시작 방법 지정
```shell
service <name> <path> [arguments]
	[option]
	...
```
- `<name>`: 서비스 이름
- `<path>`: 실행할 바이너리의 경로
- `[arguments]`: 바이너리에 전달할 추가 인수
#### 예제:
```shell
service myservice /system/bin/mybinary --arg1 --arg2
	class main
	user system
	group system
	disabled
	oneshot
```
- **옵션**:
    - `class <name>`: 서비스 클래스 지정 (예: `main`, `core`)
    - `user <username>`: 이 서비스가 실행될 사용자 지정
    - `group <groupname>`: 서비스 그룹 지정
    - `disabled`: 초기 상태에서 비활성화
    - `oneshot`: 서비스가 한 번만 실행되고 종료

### 2. **동작 섹션** (`on`)
특정 이벤트 발생 시 수행할 작업을 정의
```shell
on <trigger>
	<command>
	...
```
- `<trigger>`: 트리거 조건 (예: `boot`, `property:property_name=value`)
- `<command>`: 트리거 시 수행할 명령
#### 예제:
```
on boot
	start myservice
```

### 3. **환경 변수 정의** (`export`)
환경 변수 설정
```shell
export <name> <value>
```
#### 예제:
```shell
export PATH /sbin:/system/sbin:/system/bin:/system/xbin
```

### 4. **파일 및 디렉토리 권한 설정** (`chmod`, `chown`)
#### 예제:
```shell
chmod 0644 /data/myfile
chown system system /data/myfile
```
- `chmod`: 파일 권한 설정
- `chown`: 파일 소유자 및 그룹 설정

### 5. **속성(property) 설정** (`setprop`)
시스템 속성을 설정
```shell
setprop <property> <value>
```
#### 예제:
```shell
setprop ro.debuggable 1
```

### 6. **명령어** (`command`)
init에서 지원하는 다양한 명령어 실행
#### 주요 명령어:
- `start <service>`: 특정 서비스를 시작
- `stop <service>`: 특정 서비스를 중지
- `restart <service>`: 특정 서비스를 재시작
- `exec <path>`: 외부 명령 실행
#### 예제:
```shell
exec /system/bin/log -t mytag "Service started"
```

### 7. **조건문** (`if-else`)
조건에 따라 명령어 실행
#### 예제 1:
```shell
on property:ro.boot.mode=normal
	start normal_boot

on property:ro.boot.mode=safe
	start safe_boot
```

#### 예제 2:
```shell
on boot
	export PATH /sbin:/system/bin:/system/xbin
	chmod 0755 /data/mydir
	mkdir /data/mydir 0770 system system
	start myservice
	
service myservice /system/bin/mybinary --arg1
	class main
	user system
	group system
	disabled
```
