---
layout: post
title: "카메라/ISP V4L2 드라이버 바닥부터 ② — 다른 드라이버끼리 메모리 공유하기, 그리고 전원 끄기"
date: 2026-07-29 13:00:00 +0900
categories: driver linux-kernel dma-buf iommu runtime-pm
series: 3
description: "ftrace로 CFE→PiSP BE 사이의 dma-buf 공유를 프레임마다 실측하고, sysfs 값을 직접 200ms에서 2000ms로 바꿔가며 runtime PM 정책을 검증했다."
---

[1편]({% post_url 2026-07-29-lv3-v4l2-pipeline %})에서 raw 프레임을 뽑는 데 성공했다.
근데 실제 ISP 파이프라인은 CFE(Front End)와 PiSP BE(Back End)라는 **서로 다른 두 개의
V4L2 디바이스**로 나뉘어 있고, 이 둘이 프레임 버퍼를 복사 없이(zero-copy) 공유해야
한다. 이번 편은 그 공유 메커니즘을 ftrace로 실측하고, 안 쓸 때 전원을 어떻게 끄는지까지
다룬다.

## mmap과 뭐가 다른가

Lv1·Lv2의 zero-copy는 "커널 버퍼 → 유저스페이스 프로세스 하나"의 1:1 매핑이었다.
이번엔 "V4L2 캡처 버퍼 → **완전히 다른 커널 서브시스템**"의 공유다 — 서로 코드도
모르고 내부 구조도 모르는 두 드라이버가 같은 물리 메모리를 어떻게 합의하는가. 이 문제는
세 조각으로 쪼개진다:

| 하위 문제 | 해법 |
|---|---|
| 서로 다른 드라이버가 "같은 버퍼"라고 어떻게 합의하나 | fd 핸들(`VIDIOC_EXPBUF` → `dma_buf_export()`) |
| 각 장치의 DMA 엔진이 그 메모리를 실제로 어떻게 찾나 | IOMMU 매핑(장치마다 별도 등록) |
| 프로듀서가 다 쓰기 전에 컨슈머가 안 읽게 어떻게 막나 | fence(인터럽트 기반 완료 신호) |

## ftrace로 실제 호출 사슬을 잡다

이미 성공한 `rpicam-still` 캡처를 ftrace로 관찰해서, 이론이 말하는 흐름이 실제로
일어나는지 확인했다. 잡힌 것 중 핵심 구간:

```
vb2_dc_attach_dmabuf()                        ("나 이 버퍼 쓸래")
 └─ dma_buf_attach()
 └─ dma_buf_map_attachment_unlocked()
     └─ iommu_dma_map_sg()                    (기존 sg_table을 이 장치용으로)
         └─ iommu_map_sg()
             └─ bcm2712_iommu_map()           (진짜 레지스터 등록, 1~14회)
```

`dma_buf_attach`+`dma_buf_map_attachment_unlocked` 쌍이 뜬 직후 **마이크로초 단위로**
`iommu_dma_map_sg → iommu_map_sg → bcm2712_iommu_map`이 바로 뒤따랐고, 이게 ~33ms
간격(우리가 측정한 30fps 캡처 주기)으로 계속 반복됐다 — **매 프레임 캡처마다
CFE→PiSP BE 사이의 dma-buf 공유·IOMMU 재등록이 실제로 벌어지고 있다**는 걸 눈으로
확인한 것이다.

`bcm2712_iommu_map` 호출 횟수는 attach마다 1~14회로 들쭉날쭉했다 — 24MB짜리 프레임이
그때그때 물리 메모리에 몇 조각으로 흩어져 확보되는지(fragmentation)가 매번 달라서다.
IOMMU는 이렇게 흩어진 조각들을 장치 입장에선 하나의 연속된 주소 공간처럼 보이게
해준다.

한 가지 짐작이 틀렸었다 — Pi5는 Cortex-A76이니 IOMMU도 당연히 ARM 표준 SMMU일
거라 넘겨짚을 뻔했는데, 소스를 확인해보니 `compatible = "brcm,bcm2712-iommu"` —
**브로드컴 자체 설계**였다(`drivers/iommu/bcm2712-iommu.c`). 이 위의 레이어(dma-buf,
IOMMU-DMA 접착제)는 전부 범용 표준이지만, 진짜 레지스터를 만지는 맨 아래 함수는
언제나 벤더 고유 구현이라는 걸 다시 한번 확인한 셈이다.

## 카메라를 안 쓸 때는 전원을 어떻게 끄나

Phase A/B에서 이미 읽었던 `cfe_start_streaming()` 안에 `pm_runtime_resume_and_get()`이
있었다. "필요할 때만 켠다"는 runtime PM의 시작점이다.

**예측**: 칩 설계 시 전력이 각 블록으로 갈라져 들어가니, 그쪽 전류나 클럭을 끊으면
된다 — 정확했다. 소스를 까보니 리눅스는 `pm_runtime_get/put`(상위, "필요/불필요"만
알림) → 콜백(`SET_RUNTIME_PM_OPS`) → `clk`/`regulator`(실제 하드웨어 제어) 구조를 쓴다.
로드맵엔 이 사이에 genpd(전원 도메인 그룹)가 있을 거라 적혀 있었는데, 실제로 CFE·
PiSP BE 둘 다 자기 `runtime_suspend`/`resume` 콜백에서 `clk_disable_unprepare`/
`clk_prepare_enable`을 **직접** 호출하고 있었다 — genpd 없는 2단 구조였다(로드맵을
소스로 정정).

## sysfs로 직접 정책을 바꿔보다

읽기만 해서는 얕게 느껴져서, 실제로 값을 바꿔가며 검증했다.

```bash
cat /sys/.../pisp_be_경로/power/runtime_status
# 스트리밍 중: active / 끝난 직후: suspended
```

소스에서 PiSP BE는 `pm_runtime_set_autosuspend_delay(pispbe->dev, 200)` — 200ms
대기 후 꺼지는 걸 확인했다(CFE는 대기 없이 즉시 꺼짐 — 같은 파이프라인 안에서도
블록마다 전원 정책이 다르다). 100ms 간격으로 `runtime_status`를 찍어보니
`100~200ms: active` → `300ms: suspended`, 소스의 200ms 설정값과 정확히 일치했다.

여기서 멈추지 않고 **`power/autosuspend_delay_ms`를 직접 200 → 2000으로 바꿔서**
재검증했다:

```
100ms ~ 2000ms: active
2100ms: suspended
```

바꾼 값이 정확히 실측에 반영됐다. "값을 바꾸고 → 예측하고 → 측정해서 확인"하는
사이클을 직접 돌려본 것 — 이게 실제 BSP 업무에서 autosuspend 대기시간을 튜닝할 때
쓰는 워크플로의 축소판이다(너무 짧으면 전원 껐다켰다 반복 손해, 너무 길면 배터리
낭비).

다음 편은 이 모든 걸 숫자로 증명하는 이야기 — zero-copy가 없었다면 얼마나 손해였을지,
실제 CPU 점유율은 얼마나 낮은지, 그리고 `-O2` 컴파일러가 벤치마크 루프를 통째로
지워버렸던 사건까지.
