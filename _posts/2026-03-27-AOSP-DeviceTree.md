---
title: "[AOSP] Device Tree"
excerpt: "device Tree 란?"

categories:
  - AOSP
tags:
  - [tag1, tag2]

permalink: /AOSP/Device_Tree/

toc: true
toc_sticky: true

date: 2026-01-15
last_modified_at: 2026-04-08
---

# DTSI (Device Tree Source Include)란?
- DTSI는 Device Tree를 “모듈화”하기 위한 include 파일
```text
# 빌드 흐름
.dts + .dtsi
   ↓
dtc (device tree compiler)
   ↓
.dtb / .dtbo
   ↓
boot.img 포함
   ↓
kernel boot 시 적용
```

## 1. Device Tree 구조
```text
.dts   → 최종 보드 설정 (entry point)
.dtsi  → 공통 설정 (include 파일)
.dtbo  → overlay (동적 변경용)
```

## 2. DTSI 역할
- 1) DTSI는 **“공통 설정을 분리”**
``` 
// yupik.dtsi
#include "yupik-qupv3.dtsi"
#include "test_uart.dtsi"
```
  - SoC 기본 UART 정의 → `yupik-qupv3.dtsi`
  - Vendor 커스텀 UART → `test_uart.dtsi`

- 2) DISI 를 쓰는 이유 
```text
같은 SoC 쓰는 보드 여러 개 있을 때
common_uart.dtsi → 공통 정의
각 dts → include만 수행
```

## 3. 에제_1 (특수 노드 (UART))
```c
/ {
  aliases {
    hsuart1 = &qupv3_se10_2uart;
  };
};
```
  - `property = &label;` : 레이블이 참조하는 전체노드 경로를 문자열 특성으로 지정
  - aliases 노드의 각 속성은 다른 노드의 인덱스를 정의

## 4. 에제_2
```c
qupv3_se10_2uart: qcom,qup_uart@a88000 {
	compatible = "qcom,msm-geni-serial-hs";
	reg = <0xa88000 0x4000>;
	reg-names = "se_phys";
	interrupts = <GIC_SPI 355 IRQ_TYPE_LEVEL_HIGH>;
	clock-names = "se-clk", "m-ahb", "s-ahb";
	clocks = <&gcc GCC_QUPV3_WRAP1_S2_CLK>,
		<&gcc GCC_QUPV3_WRAP_1_M_AHB_CLK>,
		<&gcc GCC_QUPV3_WRAP_1_S_AHB_CLK>;
	pinctrl-names = "default", "active", "sleep";
	pinctrl-0 = <&qupv3_se10_default_txrx>;
	pinctrl-1 = <&qupv3_se10_2uart_active>;
	pinctrl-2 = <&qupv3_se10_2uart_sleep>;
	qcom,wrapper-core = <&qupv3_1>;
	status = "disabled";
};
```
- `compatible : "qcom,msm-geni-serial-hs"`
  - 시스템 이름 지정, "제조업체, 드라이버명" 형식의 문자열을 포함

- `reg = <0x98c000 0x4000>;`
  - 주소 지정 방법
  - 해당 node의 address와 size를 정의, \<address, size\> 형태로 구성

- `interrupts = <GIC_SPI 355 IRQ_TYPE_LEVEL_HIGH>;`
  - 디바이스의 각 인터럽트 출력 신호마다 하나씩 인터럽트 지정자 목록을 포함하는 디바이스 노드의 특성이

- `clock-names = "se-clk", "m-ahb", "s-ahb";`
  - clock 레이블

- `clocks = <&gcc GCC_QUPV3_WRAP1_S2_CLK>, <&gcc GCC_QUPV3_WRAP_1_M_AHB_CLK>,<&gcc GCC_QUPV3_WRAP_1_S_AHB_CLK>;`
  - clock 노드를 나타냄
  - 보통 clock은 수정할 일이 거의 없음.
  - 기존 kernel 파일에 있는 내용을 참고해서 추가하는 경우가 대부분
  - 괄호에 사용하는 clock에 대해 정의되어있음. (기존 kernel driver 단에 정의)

- `pinctrl-names = "default", "active", "sleep";`
  - pin 상태를 할당하는 name List
  - pinctrl은 pin mux 설정을 하고, 실제 사용할 장치 노드와 pin 노드 간의 연결 관리.
  -  장치 노드에는 고유의 compatible 이름을 가지고 있으며 이 이름은 실제 드라이버를 검색할 때 사용
  -  pinctrl-names와 pinctrl-0은 MUX 설정이며 실제 GPIO 접근을 위해서는 별도의 pin 설정을 함
  - status 가 okay가 되어야 드라이버가 활성됨.

- `pinctrl-0 = <&qupv3_se10_default_txrx>;` \
`pinctrl-1 = <&qupv3_se10_2uart_active>;` \
`pinctrl-2 = <&qupv3_se10_2uart_sleep>;`
  - pinctrl 내의 pin configuration node를 가리키는 phandles의 list

- `qcom,wrapper-core = <&qupv3_1>;`
  - Qualcomm 사의 qup v3 hardware ID

- `status = "ok";`
  - 장치의 작동 상태를 나타낸다. (ok, diswabled 등)
  - 상위 product / device 설정이 어떤 모듈을 포함할지 결정
  - make 계열 도구가 이를 모아 system/vendor/product 이미지 등을 생성하는 구조
