---
layout: post
title: "Zybo 이기종 AMP 드라이버 바닥부터 ③(완) — 커널 모듈, 캐시의 두 번째 겹, 그리고 이 칩에서 멈춘 이유"
date: 2026-07-29 11:00:00 +0900
categories: driver linux-kernel arm zynq amp remoteproc cache
series: 4
description: "SGI kick을 진짜 리눅스 커널 모듈로 옮기면서 per-CPU 인터럽트라는 새 규칙을 만났고, 캐시 실험에서는 L1 하나만으로는 안 풀리는 문제를 만났다. 그리고 결국 이 칩엔 remoteproc 드라이버가 어디에도 없다는 걸 확인하고 보드를 옮기기로 한 이야기."
---

[2편]({% post_url 2026-07-29-lv4-sgi-bidirectional-kick %})에서 SGI 양방향 kick을
U-Boot에서 손으로 증명했다. 이번 편은 그걸 진짜 리눅스 커널 모듈로 옮기고, 원래 계획에
있던 마지막 실험(cached vs non-cached 공유메모리)까지 하다가, 결국 이 칩을 떠나기로
결정한 이야기다.

## 커널 모듈로 옮기기 — SGI는 "코어별 전용" 인터럽트였다

받는 쪽(코어0)을 커널 모듈로 만들려니 새로운 API가 필요했다. 디바이스트리 `interrupts=`
속성으로는 SGI를 못 쓴다는 걸 GIC 드라이버 소스(`gic_irq_domain_translate`)에서 먼저
확인했다 — SPI/PPI 타입만 받아주는 3-cell 경로가 있고, SGI는 `param_count==1`인 별도
경로로만 매핑 가능했다. `irq_find_host()`+`irq_create_fwspec_mapping()`으로 등록.

빌드는 됐는데 `insmod`하자마자 커널이 경고를 뱉었다:

```
WARNING: ... at kernel/irq/manage.c:2183 request_threaded_irq+0x158/0x170
sgi_kick: request_irq failed (-22)
```

소스를 까보니 이유가 있었다 — `WARN_ON(irq_settings_is_per_cpu_devid(desc))`. GIC
드라이버가 hwirq 0~31(SGI+PPI 전부)을 `irq_set_percpu_devid()`로 강제 표시해두고
있었다. SGI/PPI는 태생부터 "코어별로 독립적으로 발생하는" 인터럽트라(코어0의 SGI10과
코어1의 SGI10은 완전히 별개 사건), 일반 `request_irq()`가 아니라 `request_percpu_irq()`
+ 등록 후 현재 코어에서 명시적 `enable_percpu_irq()`가 필요했다. dev_id도
`DEFINE_PER_CPU`로 코어별 저장공간을 따로 둬야 했다.

## 응답이 없다 — 이번엔 코어1이 리눅스한테 먹혔다

per-CPU 수정 후에도 여전히 반응이 없었다. `/proc/interrupts`를 열어보고서야 원인을
알았다 — **CPU1 열에 `twd`(로컬타이머) 카운트가 정상적으로 찍히고 있었다.** 즉 코어1이
우리 펌웨어가 아니라 **리눅스 커널 자신**을 실행 중이었다.

리눅스를 부팅할 때 `bootargs`에 `maxcpus=1`을 안 넣었던 게 원인이었다 — 그래서 리눅스가
자기 SMP 브링업 절차(1편에서 우리가 손으로 재현했던 것과 똑같은 트램폴린+SLCR 리셋
메커니즘)로 코어1을 정상적인 두 번째 코어로 끌고 들어가 버렸다. U-Boot에서 애써
띄워둔 우리 펌웨어는 그 순간 통째로 덮어써졌다. `maxcpus=1`을 추가하고 나서야
`insmod` 한 번에 kick과 답례가 다 찍혔다:

```
sgi_kick: reply received!
sgi_kick: loaded, kicked core1 (SGI9)
```

로그 순서를 보면 응답이 kick 완료 로그보다 **먼저** 찍혀있다 — `writel()`로 kick을 쏜
직후, 우리 코드가 마지막 로그를 찍기도 전에 코어1의 응답 인터럽트가 이미 도착해서 실행을
가로챈 것이다. 왕복이 마이크로초 단위라는 뜻이다.

## cached vs non-cached — 계획대로 안 됐다

원래 계획은 공유메모리 영역을 cached/non-cached 두 가지로 매핑해서 비교하는 것이었다.
근데 `ioremap()`도 `ioremap_cache()`도 똑같이 커널 경고와 함께 거부당했다:

```c
/* Don't allow RAM to be mapped with mismatched attributes - this
 * causes problems with ARMv6+ */
if (WARN_ON(memblock_is_map_memory(PFN_PHYS(pfn)) && mtype != MT_MEMORY_RW))
    return NULL;
```

우리 공유메모리 주소는 리눅스가 "진짜 System RAM"으로 인식하는 곳이라, 그런 주소는
**non-cached 매핑 자체가 아키텍처 차원에서 금지**돼 있었다 — 같은 물리 페이지가 서로
다른 캐시 속성으로 동시에 매핑되면 ARMv6+에서 데이터 손상까지 날 수 있어서였다. 유일하게
허용되는 건 `memremap(..., MEMREMAP_WB)`(그냥 cached).

그래서 실험을 "cached vs non-cached"에서 "flush 안 함 vs flush 함"으로 바꿨다. 근데
`flush_cache_all()`을 매번 불러도 완전히 해결이 안 됐다 — 이유는 이 함수가 **L1 캐시만**
처리하기 때문이었다(`cpu_cache.flush_kern_all`, 부팅 시 인식한 CPU 모델 자신의 내장
캐시 함수). Zynq-7000엔 이것과 별개로 **L2 캐시(PL310, 외장 컨트롤러)**가 하나 더
있는데, 이건 `outer_cache`라는 완전히 다른 인터페이스로 관리되고, **그 심볼들은
로더블 모듈한테 아예 공개(export)돼 있지 않았다.** L1은 flush할 수 있어도 L2는 손을
못 대는 상황이었다.

이 실험이 알려준 건 결국 이거였다 — "flush로 땜빵"은 여러 캐시 레벨이 있는 실제
하드웨어에선 근본적으로 불완전하고, 진짜 정답은 애초에 그 영역을 캐시가 안 타게
설계하는 것(디바이스트리 `reserved-memory`로 일반 RAM 풀에서 미리 빼두는 것)이다.

## 이 칩엔 remoteproc가 없다

여기까지 오고 나서 remoteproc 프레임워크로 넘어가려는데, mainline 커널 소스를 뒤져도
Zynq-7000(듀얼 Cortex-A9)용 remoteproc 드라이버가 없었다. "그럼 Xilinx 벤더 커널엔
있겠지"라고 생각하고 `linux-xlnx`(Xilinx 자체 관리 커널, 최신 브랜치)를 새로 클론해서
확인했는데, **거기도 없었다.** 있는 건 `xlnx_r5_remoteproc.c` 하나뿐인데, 이건 완전히
다른 칩(ZynqMP, Cortex-A53+R5)용이었다.

Xilinx가 최근엔 ZynqMP·Versal 쪽에 집중하면서, 한때 있었을 Zynq-7000 AMP 지원(2017~2018년
경 SDK 시절 레퍼런스 디자인)이 정식 커널 트리 밖에만 존재했거나 이후 정리된 것으로
보인다.

선택지는 셋이었다 — ① 벤더 커널(없으니 제외) ② 우리가 직접 remoteproc 드라이버를 짠다
③ 다른 보드로 옮긴다. ②의 매력은 분명했다 — 이 세 편에서 한 작업(SLCR 제어, 트램폴린,
SGI 송수신)이 사실 remoteproc의 `.start`/`.stop`/`.kick` 콜백 본체 그대로였으니까.
근데 문제는 남은 20%(프레임워크 등록 규약)가 아니라, **참조 구현이 전혀 없는 상태에서
검증해야 한다는 것**이었다. 이번 시리즈에서 겪은 버그들(spin-table 오판, SLCR 잠금,
per-CPU IRQ)은 전부 리눅스 자신의 다른 코드와 대조해가며 풀 수 있었는데, remoteproc
프레임워크 자체의 규약 버그엔 그런 대조군이 없다.

Lv4의 진짜 목표는 "리눅스에 부하를 걸어도 실시간 코어는 영향 없다"를 증명하는 것이지
드라이버 저작 자체가 아니었다. **③ 보드 전환을 택했다** — mainline이 remoteproc/rpmsg를
정식 지원하는 STM32MP157F-DK2(Cortex-A7+M4)로. 이 세 편에서 만든 것들
(`lv4/` 디렉토리)은 지우지 않았다 — remoteproc가 내부적으로 뭘 하는지 레지스터
레벨로 이해한 채로 다음 보드의 프레임워크를 쓰게 됐으니, 이 실험들은 헛수고가 아니라
그 자체로 완결된 하나의 이야기다.

다음 챕터는 새 보드에서 이어진다.
