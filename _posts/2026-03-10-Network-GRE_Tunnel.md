---
title: "[Network] GRE Tunnel"
excerpt: "GRE Tunnel 이란?"

categories:
  - Network
tags:
  - [tag1, tag2]

permalink: /Network/GRE_Tunnel/

toc: true
toc_sticky: true

date: 2026-03-17
last_modified_at: 2026-04-06
---

# 1. GRE 터널 유형

| 유형                                     | 설명                                           | 특징                           |
| -------------------------------------- | -------------------------------------------- | ---------------------------- |
| **Transparent Tunnel<br>(IP-over-IP)** | 원본 패킷 그대로 캡슐화<br>별도 내부 IP 없음                 | 브리지형 터널<br>내부 IP 없이 단순 전달    |
| **Routed Tunnel <br>(Point-to-Point)** | GRE 인터페이스에 자체 IP 부여(/30 등)<br>라우팅을 통해 내부망 전달 | 가상 라우터 역할<br>경로 제어 가능<br>안정적 |

## Transparent Tunnel (IP-over-IP, 투명 터널)
```c
GRE1 (192.168.179.38) <--> GRE1 (192.168.179.133)
   │                           │
192.168.10.10             192.168.20.10
```
## Routed Tunnel (Point-to-Point 링크형 터널)
```c
[192.168.100.1] --- GRE 터널 --- [192.168.100.2]
        ↑                             ↑
192.168.179.38                   192.168.179.133
(wlan0, 실제 IP(192.168.10.10))   (wlan0, 실제 IP(192.168.20.10))
```

# 2. Transparent GRE 터널 구성 예시
### 송신 (wlan0: 192.168.179.38, 스마트폰: 192.168.10.10)
```c
# GRE 터널 생성
ip tunnel add gre1 mode gre local 192.168.179.38 remote 192.168.179.133
ip link set dev gre1 up

# 송신 패킷을 GRE 인터페이스로 라우팅
ip route add 192.168.20.0/24 dev gre1
```
### 수신 (wlan0: 192.168.179.133, 스마트폰: 192.168.20.20)
```c
# GRE 터널 생성
ip tunnel add gre1 mode gre local 192.168.179.133 remote 192.168.179.38
ip link set dev gre1 up

# 송신 패킷을 GRE 인터페이스로 라우팅
ip route add 192.168.10.0/24 dev gre1
```

# 3. GRE 터널 구성 예시
### 송신 측 (wlan0: 192.168.179.38, 스마트폰: 192.168.10.10)
```c
# GRE 터널 생성
ip tunnel add gre1 mode gre local 192.168.179.38 remote 192.168.179.133
ip link set dev gre1 up

# GRE 인터페이스에 point-to-point IP 할당
ip addr add 192.168.100.1/30 dev gre1

# 터널을 통해 상대 내부망으로 라우팅
ip route add 192.168.20.0/24 via 192.168.100.2
```
### 수신 측 (wlan0: 192.168.179.133, 스마트폰: 192.168.20.20)
```c
# GRE 터널 생성
ip tunnel add gre1 mode gre local 192.168.179.133 remote 192.168.179.38
ip link set dev gre1 up

# GRE 인터페이스에 point-to-point IP 할당
ip addr add 192.168.100.2/30 dev gre1

# 터널을 통해 상대 내부망으로 라우팅
ip route add 192.168.10.0/24 via 192.168.100.1
```

### 구조
```text
[송신 내부망] 192.168.10.0/24
       │
      gre1 (192.168.100.1)
       │   GRE 캡슐화
       ▼
      wlan0 (192.168.179.38) → 인터넷
       │
[수신 wlan0] 192.168.179.133
       │
      gre1 (192.168.100.2)
       │
[수신 내부망] 192.168.20.0/24
```
- `via 192.168.100.x` → GRE 내부 point-to-point 주소를 통해 라우팅
- `dev gre1` 만 사용할 수도 있지만, 안정성과 디버깅을 위해 `via` 추천

# 4. GRE 터널 관리 명령
### 생성/삭제
``` bash
# 터널 삭제
ip link delete gre1

# 생성
ip tunnel add gre1 mode gre local <local_IP> remote <remote_IP>
ip link set dev gre1 up
```
### 상태 확인
``` bash
ip link show gre1
ip addr show dev gre1
ip route show

# 포워딩 설정
cat /proc/sys/net/ipv4/conf/gre1/forwarding
```
### 트래픽 확인 (tcpdump)
``` bash
tcpdump -i gre1 -n

# GRE 패킷 캡처 (모든 인터페이스)
tcpdump -i any 'ip proto 47'

# 특정 인터페이스만 dump
tcpdump -i wlan0 'ip proto 47'
tcpdump -i gre1 'ip proto 47'

# 출발지(source) 또는 목적지(destination)가 `<IP>`인 패킷만 필터링
tcpdump -i any host <IP>
```
### 라우트 제거
```
ip route del 192.168.20.0/24 via 192.168.100.2
```

# 5. 디버깅 순서
1. **GRE 인터페이스 활성화 확인**
``` bash
ip link show gre1
```
2. **터널 내부 IP 확인**
``` bash
ip addr show dev gre1
```
3. **라우팅 확인**
``` bash
ip route show
```
4. **패킷 캡처**
``` bash
tcpdump -i any 'ip proto 47'
tcpdump -i gre1 -n
```
5. **ping 테스트**
``` bash
ping 192.168.100.2   # tunnel point-to-point ping
ping 192.168.20.10   # 수신 내부망 ping
```

# 6. 주의 사항
1. **같은 서브넷 사용 금지**
    - 송신/수신 내부망이 동일 서브넷이면 `dev gre1` 라우트만으로는 GRE가 정상 동작하지 않음
    - 내부망이 겹치면 브리지형(gretap) 사용 필요 (2 Layer 까지 올라가야함.)
        
2. **Android 환경**
    - rp_filter, ip_forward 활성화 필요
    - 일부 커널에서는 인터페이스에 IP 없으면 GRE decap 실패 가능 → /30 IP 할당 권장

3. **디버깅**
    - 터널 IP를 ping → 캡슐화/디캡슐 확인
    - tcpdump로 GRE 패킷 수신 확인
