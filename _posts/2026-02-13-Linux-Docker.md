---
title: "[Linux] Docker"
excerpt: "Docker 란?"

categories:
  - Linux
tags:
  - [tag1, tag2]

permalink: /Linux/Docker/

toc: true
toc_sticky: true

date: 2026-02-13
last_modified_at: 2026-03-27
---

# Docker 란?
> PC 커널을 공유하면서, 완전히 분리된 Linux 실행 공간을 하나 만드는 기술

## Image (이미지)
> 설치된 OS + 프로그램 + 설정을 통째로 묶어둔 설계도
- `ubuntu:16.04` : Ubuntu 16.04 유저 공간 + 기본 패키지 상태
- 특징
	- 읽기 전용
	- 변경 불가
	- 여러 컨테이너가 같은 이미지 공유

## Container (컨테이너)
> 이미지를 실제로 실행한 인스턴스
- `docker run ubuntu:16.04` : ubuntu:16.04 이미지로부터 하나의 실행 환경을 띄움
- 특징
	- 실행 중인 Linux 프로세스 집합
	- 컨테이너마다 파일시스템 / 프로세스 / 네트워크 분리
	- 삭제해도 이미지 영향 없음

## Dockerfile
> : 이미지를 만드는 레시피

``` dockerfile
FROM ubuntu:16.04
RUN apt-get install -y openjdk-8-jdk
```
- 의미
	- “Ubuntu 16.04 기반으로”
	- “JDK 8 설치된 이미지 만들어줘”

# 기본적인 명령어 
**docker [대상] [액션]**  
→ [대상] : container(생략 가능), image, volume, network 등  
→ [액션] : ls, inspect, start, run 등

- 이미지 목록 보기 
  `sudo docker images`
- 이미지 받기
  `sudo docker pull`
- 이미지 삭제
  `sudo docker rmi [이미지 id]`
- 컨테이너 목록 보기
  `sudo docker ps`
- 컨테이너 실행 
  `sudo docker run [options] image[:TAG|@DIGEST] [COMMAND] [ARG..]`
- 컨테이너 시작
  `sudo docker start [컨테이너 id / name]`
- 컨테이너 정지
  `sudo docker stop [컨테이너 id / name]`
- 컨테이너 삭제
  `sudo docker rm [컨테이너 id / name]`
- 컨테이너명 변경
  `sudo docker rename [기존이름] [변경할이름]`

```
sudo docker run --name project_ubuntu_16.04 -it -d --hostname jung
```
