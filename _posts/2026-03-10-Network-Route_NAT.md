---
title: "[Network] Route 및 NAT"
excerpt: "Routing 및 NAT(Network Address Translation)에 관하여"

categories:
  - Network
tags:
  - [tag1, tag2]

permalink: /Network/Route_NAT/

toc: true
toc_sticky: true

date: 2026-03-12
last_modified_at: 2026-04-05
---

# 1. IP forward
> 다른 네트워크로 패킷을 전달(Forward) 가능, **라우터** 역할 수행

- 해당 장비 전체에서 IP forwarding 허용
```shell
echo 1 > /proc/sys/net/ipv4/ip_forward
```
  - `ip_forward` : forward 활성/비활성

- iptables로 interface별 제어
```shell
iptables -A FORWARD -i eth0 -o eth1 -j ACCEPT
iptables -A FORWARD -j DROP
```
  - `-i eth0 -o eth1` : eth0 → eth1 만 허용
  - `-j DROP` : 그외 전부 차단

# 2. ip rule 
> 어떤 패킷이 어떤 라우팅 테이블을 사용할지 결정하는 규칙

- 예시 1)
```shell
ip rule add from all lookup main
```
  - `from all` : 출발지 ip가 all (즉, 모든 IP 주소)인 패킷에 대하여
  - `lookup main` : 리눅스 시스템에서 기본적으로 사용되는 주요 라우팅 테이블을 조회

- 예시 2)
```shell
ip rule add from 192.168.1.2 table eth0
```
  - `from 192.168.1.2`:  192.168.1.2에서 출발하는 패킷은
  - `table eth0` : eth0 이름의 라우팅 테이블 사용

# 3. IP route
> 패킷의 목적지에 따른 경로 선택
```
ip route add [목적지 네트워크] via [게이트웨이 주소] dev [인터페이스 이름]
```

- 예시 1) `192.168.1.0/24` 네트워크로 가는 패킷이 `192.168.0.1` 게이트웨이를 통해 `eth0` 인터페이스를 사용해 전송되도록 라우팅 경로 추가
```shell
ip route add 192.168.1.0/24 via 192.168.0.1 dev eth0
```
  - `192.168.100.0/24` : 목적지 IP 범위(192.168.100.0~192.168.100.255)까지
  - `dev eth0` : 해당 트래픽을 eth0 인터페이스를 통해 전송

- 예시 2)
``` shell
ip route add default via 192.168.1.1 dev eth0 table 1234 src 192.168.1.2
```
  - `ip route add`: 라우팅 경로를 추가
  - `default`: 기본 경로를 의미(목적지 주소가 명시되지 않은 모든 패킷 해당)
  - `via 192.168.1.1` : 패킷을 보낼 게이트웨이(라우터)의 주소
  - `dev eth0` : eth0 인터페이스를 통해 경로를 사용
  - `table 1234` : 1234 라는 사용자 정의 라우팅 테이블에 경로를 추가
  - `src 192.168.1.2` : 출발지 IP 주소

# 4. iptables
> 패킷 필터링 및 방화벽 관리를 위한 도구

- 현재 선택된 체인(기본적으로 INPUT, FORWARD, OUTPUT 체인)에서 모든 규칙을 삭제
```shell
iptables -t nat -F
iptables -F
```

# 5. NAT(Network Address Translation)
> 패킷의 IP 주소(또는 포트)를 변경하는 기술

## 1) 패킷의 흐름
```
[패킷 들어옴]
      ↓
PREROUTING  → DNAT(Destination NAT) 여기서 발생
      ↓
Routing 결정
      ↓
FORWARD / INPUT
      ↓
POSTROUTING → SNAT(Source NAT) 여기서 발생
      ↓
[패킷 나감]
```
- `SNAT (Source NAT)` : 출발지 IP 변경
- `DNAT (Destination NAT)` : 목적지 IP 변경

## 2) PREROUTING
> 네트워크 패킷이 라우터나 방화벽과 같은 네트워크 장비에 도착했을 때, 패킷이 목적지에 도달하기 전에 적용되는 규칙

- PREROUTING 예시(1) - Destination IP 변경
```shell
iptables -t nat -A PREROUTING -i eth0 -j DNAT --to 192.168.100.10
```
  - `-i eth0` : eth0 interface를 통해 들어온 패킷
  - `-j DNAT --to 192.168.100.10` : destination IP를 192.168.100.10으로 변경
 
- PREROUTING 예시(2) - TCP, UDP 패킷만 허용
```shell
iptables -t raw -A PREROUTING -p tcp --dport 5201 -j ACCEPT
iptables -t raw -A PREROUTING -p tcp --sport 5201 -j ACCEPT
iptables -t raw -A PREROUTING -p udp --dport 5201 -j ACCEPT
iptables -t raw -A PREROUTING -p udp --sport 5201 -j ACCEPT
```
  - `-p` : TCP, UDP protocol (프로토콜)
  - `--dport` : destination port
  - `--sport` : source port
  - `-j` : Accept 의미

## 3) POSTROUTING
> 패킷이 나가기 직전 단계에서 적용되는 규칙

- 동적IP 예시 : 패킷 송신 시 Source IP를 변경 (동적)
```shell
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```
  - `-t nat`: NAT 테이블을 사용
  - `-A POSTROUTING`: 패킷이 나가기 직전 단계에서
  - `-o eth0`: 외부로 나가는 인터페이스가 `eth0`일 때
  - `-j MASQUERADE`: 출발지 주소를 해당 인터페이스의 공인 IP로 변환

- 고정IP 예시 : 패킷 송신 시 Source IP를 192.168.100.1로 변경 (정적)
```shell
iptables -t nat -A POSTROUTING -o eth0 -j SNAT --to 172.31.93.xxx
```
  - `-t nat`: NAT 테이블을 사용
  - `-A POSTROUTING`: 패킷이 나가기 직전 단계에서
  - `-o eth0`: 외부로 나가는 인터페이스가 `eth0`일 때
  - `-j SNAT --to 192.168.100.1`: 출발지 주소를 공인 IP(172.31.93.xxx)로 변환

