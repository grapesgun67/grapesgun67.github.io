---
layout: post
title: "RPi5 하이퍼바이저 격리 드라이버 바닥부터 ④ — 최악의 결과가 설정 하나로 뒤집히다"
date: 2026-08-12 21:00:00 +0900
categories: driver linux-kernel iommu smmu kvm isolation automotive
series: 6
description: "SPI에 IOMMU가 없어서 패스스루는 원천 차단, 대안(ivshmem)을 저비용으로 먼저 검증했더니 P1 최악값보다 4~10배 나쁜 결과가 나왔다. chrt 한 줄로 정반대로 뒤집힌 이야기."
---

Lv5 P3는 이 프로젝트 전체에서 "★최대 리스크 관문"이라고 미리 못박아둔 단계다 — 텔레메트리
취득 경로(SPI)를 세 가지 아키텍처((A) SPI 패스스루 / (B) Host 취득+ivshmem / (C) 소비
태스크만 격리) 중 어디에 배치할지 결정해야 하는데, 이 결정이 [4편]({% post_url
2026-08-12-lv5-kvm-showcase-tcg-vs-kvm %})까지 쌓은 KVM 인프라 위에서 P4~P5 전체 설계를
좌우한다. 상한 2주로 못박고 시작했다.

## (A) SPI 패스스루 — 실측 한 방으로 원천 차단

```bash
dmesg | grep -iE "iommu|smmu"
ls /sys/kernel/iommu_groups/
find /sys/bus/platform/devices/ -iname "*spi*"
```

BCM2712의 IOMMU 드라이버는 `bcm2712-iommu`뿐이고 `arm,smmu` 계열은 전혀 없다 —
[Lv3에서 카메라 파이프라인 조사할 때]({% post_url 2026-07-29-lv3-dmabuf-runtime-pm %})
이미 확인했던 "표준 ARM SMMU가 아니라 Broadcom 자체 설계"라는 결론과 일치한다. IOMMU
그룹은 셋뿐이고 붙어있는 장치는 코덱/ISP/GPU/디스플레이 합성기/카메라 — 전부 그래픽·
카메라 계열이다. **SPI 컨트롤러는 dmesg의 IOMMU 관련 줄 어디에도 안 나오고,
`iommu_group` sysfs 노드 자체가 없다** — 어떤 IOMMU도 안 거치는 raw DMA 경로라는 뜻.

**IOMMU 자체가 없으면 SMMU 표준 준수 여부를 따질 것도 없이 vfio-platform이 바인드할
방법이 없다.** (A)는 여기서 확정적으로 닫혔다.

이 판단이 임의가 아니라는 걸 웹 리서치로 한 번 더 확인했다 — Intel의 자동차 IVI용
오픈소스 하이퍼바이저 **ACRN**의 공식 문서가 "VT-d(IOMMU) 미지원 플랫폼은 passthrough를
포기하고 emulation/para-virtualization으로 전환해야 한다"는 하드 룰을 명시하고 있었다.
업계 표준 하이퍼바이저도 같은 로직을 쓴다는 근거다. 흥미로운 학술 근거도 하나 더
있었다 — `Beyond the Bermuda Triangle of Contention`(2025)이라는 논문이 "IOMMU가 장치
단위 접근은 완벽히 격리해도 여러 장치가 같은 메모리버스/캐시를 공유하는 한 시간적
간섭은 못 막는다"고 지적하는데, 이게 [3편]({% post_url
2026-08-11-lv5-rt-baseline-honest-conclusion %})에서 실측으로 발견한 것과 정확히 같은
결론이다 — 이 프로젝트만의 특이 현상이 아니라 학계가 이름까지 붙여 연구 중인 잘 알려진
문제였다.

## (B) 저비용 사전검증 — 1차 결과는 재앙이었다

(A)가 막혔으니 (B) Host 취득+ivshmem으로 방향을 잡되, P4의 무거운 ivshmem 파이프라인을
다 만들기 전에 **싸게 먼저 검증**하기로 했다. 기존 P1/P2 인프라를 재사용해서: Host는
`isolcpus=3 nohz_full=3`, 게스트(kvmtool, 1 vCPU)의 `kvm-vcpu-0` 스레드를 `taskset`으로
CPU3에 핀, Host CPU0~2엔 `stress-ng` 부하, 게스트 안에서 `cyclictest -p99`.

**1차 결과가 심각하게 나빴다.** Max가 3.9~9.7ms — P1의 최악 조건보다도 4~10배 나쁜
숫자였다. `perf kvm stat live`로 봐도 `WFx`(idle 트랩) 평균이 44~52us 수준이라 이 정도
스파이크를 설명하지 못했다 — VM exit 비용이 원인이 아니었다.

## 원인 진단 — 게스트 안의 RT는 호스트엔 안 통한다

`/proc/interrupts`를 보니 CPU3에 `IPI1`(Function call interrupts — TLB shootdown 등
코어 간 강제 동기화)이 1720회 찍혀있었다. P1에서 이미 "isolcpus가 못 막는 채널"로
지목했던 바로 그것이다. 그런데 결정적인 확인은 따로 있었다 — `chrt -p <kvm-vcpu-0 TID>`
를 쳤더니 **`SCHED_OTHER`, 우선순위 0**이 나왔다.

즉 게스트 안에서 cyclictest가 요청한 `-p99`(SCHED_FIFO) RT 우선순위는 **게스트 내부
에서만 유효**했다. 그 게스트 전체를 실제로 돌리는 호스트 스레드(`kvm-vcpu-0`)는 호스트
스케줄러 입장에서 그냥 평범한 프로세스였던 것 — TLB shootdown 같은 호스트 커널의 급한
작업이 아무 망설임 없이 그 스레드를 밀어냈다. **게스트 내부 RT 설정은 호스트 쪽 vCPU
스레드에 자동으로 전파되지 않는다**는, 실제 프로덕션 하이퍼바이저 운영에서도 흔히
놓치는 함정을 그대로 밟은 것이다.

## chrt 한 줄로 완전히 뒤집히다

`sudo chrt -f -p 90 <kvm-vcpu-0 TID>`로 호스트 쪽 vCPU 스레드도 SCHED_FIFO로 직접
승격시키고 4회 반복 측정했다.

| 시도 | Max |
|---|---|
| 1 | 515us |
| 2 | 169us |
| 3 | 80us |
| 4 | 119us |

평균 220.75±199.5us, 최악 515us — **P1 최고 조건(`rt-isolcpus-nohz`, 평균 364.3±134.5us,
최악 515us)과 동등하거나 더 나은 결과**로 뒤집혔다. n=4라 P1만큼의 반복 횟수는 아니지만,
"KVM 게스트 경계가 격리에 해롭다"에서 "제대로 세팅하면 순수 코어 격리와 동등 이상"으로
결론이 완전히 바뀌었다. **설정 한 줄이 4~10배 나쁜 결과를 P1 최고 조건과 맞먹는 결과로
바꿔놓은 셈**이다 — P4 설계에 "호스트 쪽 vCPU 스레드에 RT 우선순위를 주는 게 필수"라는
요건을 반드시 반영하기로 했다.

## MPAM은 없었다 — 정직하게 남겨두는 한계

P1에서 발견한 "코어 격리로는 메모리버스 경합을 못 막는다"는 구멍을 진짜로 메우는 업계
해법은 `H-MBR`(하이퍼바이저 레벨 메모리 대역폭 예약)이고, 이게 기대는 하드웨어 기능이
**ARM MPAM**(Memory Partitioning And Monitoring)이다. 그래서 이것도 실측했다:

```bash
dmesg | grep -i mpam            # 빈 결과
ls /sys/fs/resctrl              # No such file or directory
cat /proc/cpuinfo | grep Features  # mpam 플래그 없음
```

네 가지 확인(dmesg/resctrl 마운트/커널 config/CPU feature 플래그) 전부 일관되게 없었다
— **BCM2712/Cortex-A76은 MPAM을 지원하지 않는다.** 부분 지원도 아니고 완전히 없다. 이
문제를 하드웨어 레벨로 깔끔하게 풀 방법이 이 칩엔 없다는 뜻이라, "부하가 있어도 완벽히
flat"이 아니라 "격리로 최선을 다했지만 이 하드웨어의 근본적 한계가 있다"는 정직한
프레이밍을 [5편(P5 플래그십 실험)]({% post_url 2026-08-19-lv5-isolation-proof-flagship
%})까지 이어가기로 했다.

자동차 업계가 이 종류의 리스크를 실제로 얼마나 진지하게 다루는지도 확인했다 — 게스트
VM이 SMMU 폴트를 일으켰을 때 그 게스트만 격리/종료시켜 나머지 시스템을 지키는
메커니즘을 다룬 특허가 존재했다. "게스트가 SMMU/IOMMU 경계를 잘못 건드릴 수 있다"는
위험이 자동차 하이퍼바이저 업체들이 특허까지 내며 방어하는 진짜 리스크라는 뜻이고,
P3에서 SMMU 검증을 먼저 하고 넘어가는 신중한 접근이 업계 감각과 맞아떨어진다는 근거였다.

## 최종 결정

| 갈래 | 판정 | 근거 |
|---|---|---|
| (A) SPI 패스스루 | ❌ 불가 | SPI에 IOMMU 자체가 없음(실측) — ACRN 공식 룰과도 일치 |
| **(B) Host 취득 → ivshmem** | **✅ 채택** | 저비용 사전검증 결과 P1 최고 조건과 동등 이상(단, 호스트 vCPU 스레드 RT 우선순위 필수) |
| (C) 소비 태스크만 격리 | 보류 | (B)가 이미 검증됐고 격리 강도가 더 약해서 우선순위 낮음 |

(B)가 그냥 차선책이 아니라는 근거도 하나 더 있었다 — 최신 연구와 Automotive Linux
Foundation의 VirtIO 센서 표준화 작업 논조가 "공유메모리(ivshmem) 설계가 순수 virtio
링버퍼보다 안전·지연 critical한 자동차 도메인에 더 적합하다" 쪽으로 가고 있었다. P4에서
가려는 `QEMU ivshmem-plain` 방향이 업계가 실제로 밀고 있는 방향과 일치한다는 뜻이다.

## 다음

P4는 이 결정을 실제 파이프라인으로 만드는 단계다 — QEMU ivshmem-plain에 게스트 커널
드라이버를 직접 짜서 붙이고, 오늘 얻은 "vCPU 스레드 RT 우선순위 필수"라는 설계 요건을
반영해서 Nucleo→SPI→ivshmem→게스트까지 zero-copy로 잇는다.
