---
layout: post
title: "STM32MP157F-DK2로 넘어와서 ④(완) — Yocto 레이어로 패키징하고, 두 접근을 비교하다"
date: 2026-08-09 09:00:00 +0900
categories: driver linux-kernel yocto bitbake amp remoteproc
series: 5
description: "손으로 크로스빌드하던 커널 모듈을 정식 Yocto 레이어로 옮기면서 겪은 함정들, 그리고 Zynq(레지스터 직접 제어)와 STM32MP1(mainline 프레임워크) 두 접근이 사실은 같은 문제를 풀고 있었다는 이야기로 Lv4를 마무리한다."
---

[3편]({% post_url 2026-08-07-lv4mp1-adc-jitter-proof %})까지 Lv4의 핵심 실험(부하 중
M4 지터 불변)을 끝냈다. 지금까지 `rpmsg_latency.ko`/`rpmsg_virtadc.ko`는 전부 손으로
크로스빌드해서 `scp`로 올린 것들이었다 — 이번 편은 그걸 정식 Yocto 커스텀 레이어로
패키징하고, Zynq와 STM32MP1 두 여정을 비교하면서 Lv4를 마무리한다.

## Yocto 레이어의 구조 — Buildroot와 다른 3단 구조

Buildroot는 보통 "패키지 하나 = 파일 하나"로 끝나지만, Yocto는 **레이어(컨테이너) →
recipe(개별 패키지) → bbappend(기존 recipe에 조각 붙이기)** 3단 구조다. `bitbake-layers
create-layer meta-lv4mp1`로 레이어를 만들고, 그 안에 커널 모듈 recipe 하나를 넣었다:

```bitbake
FILESEXTRAPATHS:prepend := "${THISDIR}/../../../driver:"

SRC_URI = "file://src/rpmsg_latency.c \
           file://src/rpmsg_virtadc.c \
           file://include/virtadc_uapi.h \
           file://Makefile"
S = "${WORKDIR}"

inherit module
```

`module.bbclass`를 상속하면 `KDIR`/`CROSS_COMPILE` 연결(2편에서 손으로 다 하던 그 일)을
프레임워크가 대신 처리해준다 — 다만 그 위안은 오래가지 않았다.

## 세 가지 함정

**1. `SRC_URI`의 서브경로.** `file://<이름>`은 `FILESEXTRAPATHS` 디렉토리 바로 안에서
그 이름 그대로 찾는다. `.c` 파일은 `driver/src/`에 있는데 파일명만 적어서 처음엔 fetch가
실패했다 — `file://src/rpmsg_latency.c`처럼 서브경로를 명시해야 했다.

**2. `LIC_FILES_CHKSUM`.** OE는 "이 라이선스를 확인했다"는 증거로 실제 파일의 체크섬을
recipe에 박아두는데, 커널이 배포하는 `COPYING`은 GPL 전문이 아니라 어디를 보라는
포인터 파일뿐이었다. OE-Core가 자체적으로 갖고 있는 표준 라이선스 전문
(`${COMMON_LICENSE_DIR}/GPL-2.0-only`)을 직접 가리키는 게 맞는 방법이었다.

**3. 진짜 원인 — `modules.order`를 스스로 지우고 있었다.** 커널 모듈 recipe 전체
(fetch→unpack→compile→install→package)가 에러 없이 끝났는데, 정작 만들어진 `.deb`
안에 `.ko`가 없었다. 원인은 우리 기존 `Makefile`의 `modules` 타겟 — 빌드 끝나고
`modules.order`/`Module.symvers`를 정리(`rm`)하고 있었다. 손빌드 시절엔 무해한 뒷정리
였지만, Yocto의 `do_compile`(=`modules` 타겟) 다음에 곧바로 `do_install`(=새로 추가한
`modules_install` 타겟)이 실행되는 구조에서는 — 외부 커널 모듈의 `modules_install`이
"이번에 뭘 빌드했는지" 알아내는 데 쓰는 바로 그 `modules.order`를 먼저 지워버리는
꼴이었다. kbuild는 설치할 목록이 없으면 **에러 없이 그냥 0개를 설치하고 조용히
끝난다** — 이래서 태스크는 매번 "성공"으로 뜨는데 실제 산출물은 없는, 가장 헷갈리는
종류의 실패였다. 그 `rm` 줄을 빼는 것으로 해결.

## 실물 검증 — 손빌드본과 동일하게 동작하는가

레시피가 만든 이미지를 실제로 보드에 재플래싱하고 확인했다:

```
root@stm32mp1:~# modprobe rpmsg_latency
[  292.578047] rpmsg_latency: loading out-of-tree module taints kernel.
root@stm32mp1:~# modprobe rpmsg_virtadc
root@stm32mp1:~# echo OpenAMP_TTY_echo_wakeup_CM4.elf > /sys/class/remoteproc/remoteproc0/firmware
root@stm32mp1:~# echo start > /sys/class/remoteproc/remoteproc0/state
[ 1029.548659] rpmsg_virtadc virtio0.rpmsg-virtadc.-1.1026: virtadc: /dev/virtadc0 ready
```

M4 부팅부터 RPMsg 채널 3개 생성, `/dev/virtadc0` 생성까지 — 손으로 크로스빌드하던
모듈과 완전히 동일하게 동작했다.

## CI — 전체 Yocto 빌드가 아니라 컴파일 검증만

GitHub Actions에 전체 Yocto 이미지를 매번 새로 빌드하는 건 시간/비용이 너무 크다.
대신 스코프를 좁혔다 — mainline 커널 tarball을 받아 `modules_prepare`(전체 빌드가
아니라 외부 모듈 빌드에 필요한 최소한만 준비)만 돌리고, 그 위에서 `arm-linux-gnueabihf`
툴체인으로 우리 `.c` 파일이 여전히 컴파일되는지만 확인한다. 커널의 자체 Makefile을
바깥에서 `make -C <커널트리> M=<모듈디렉토리> modules`로 호출하면, `driver/Makefile`의
`obj-m` 줄만 읽고 나머지(Yocto 전용 타겟들)는 무시하기 때문에, 이 CI는 Yocto 전용
로직을 하나도 안 건드리고도 "적어도 소스는 안 깨졌다"는 가장 저렴한 안전망 역할을 한다.

## Zynq vs STM32MP1 — 결국 같은 문제였다

챕터 4(Zynq)와 이번 챕터를 나란히 놓고 보면:

| 항목 | Zynq-7000 | STM32MP1 |
|---|---|---|
| remoteproc 드라이버 | 없음 (직접 확인) | mainline 표준 채택 |
| 코어 기동 | SLCR reset-hold + 트램폴린 직접 구현 | 프레임워크가 자동 처리 |
| IPC 시그널링 | GIC SGI 레지스터 직접 조작 | `stm32-ipcc` 표준 mailbox |
| 데이터+시그널 | 공유 SRAM + SGI를 손수 분리 설계 | vring + IPCC (표준 virtio-rpmsg) |

겉보기엔 완전히 다른 두 구현이지만, 둘 다 결국 **"공유메모리(데이터) + 인터럽트
(시그널)"라는 동일한 2단 패턴**으로 수렴했다. Zynq의 SGI 왕복(챕터 4 ②편)이 사실은
이번 챕터의 RPMsg/vring 메커니즘을 손으로 먼저 만들어본 것과 다르지 않았다는 걸,
두 플랫폼을 다 거치고서야 뒤늦게 깨달았다. OpenAMP이 크로스벤더 표준(Linux Foundation
프로젝트)인 이유도 여기 있다 — 두 칩이 푸는 문제의 정체성 자체는 같으니까.

TI Jacinto, NXP i.MX, Siemens의 자동차 mixed-criticality 사례 등 실제 프로덕션은
전부 STM32MP1 쪽 패턴(mainline remoteproc+RPMsg+OpenAMP)을 쓴다. 그렇다고 Zynq
작업이 무의미한 건 아니다 — 벤더가 remoteproc을 안 만들어준 칩을 실무에서 만났을 때,
그 프레임워크가 내부적으로 뭘 하는지 이미 손으로 재현해본 경험이 있으면 그 갭을
스스로 메울 수 있다.

## Lv4를 마치며

P0(부트체인 설계)부터 여기까지, 표준 프레임워크를 실전에서 쓸 줄 아는 것(커널 client
드라이버 자작, 지연 실측, 부하 중 지터 불변 실증, Yocto 정식 패키징)과, 그 프레임워크가
없을 때 내부 메커니즘을 이해하고 손으로 재현할 수 있는 것(SLCR/GIC/캐시 코히런시를
소스 레벨로 직접 확인) — 이 둘을 같이 갖춘 게 이번 Lv4의 결론이다. 다음은 Lv5,
RPi5 기반 하이퍼바이저+TrustZone이다.
