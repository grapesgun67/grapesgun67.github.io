---
layout: post
title: "QEMU로 PCIe 가속기 드라이버 바닥부터 ① — edu 장치 해부"
date: 2026-07-09 09:00:00 +0900
categories: driver qemu pcie
series: 1
description: "BSP/디바이스 드라이버 포트폴리오: QEMU edu 장치의 BAR/MMIO, INTx vs MSI, DMA 레지스터 구조를 실제 소스로 해부한다."
---

## 왜 QEMU인가

BSP/디바이스 드라이버 채용 공고를 여러 개 분석해보면 공통 뼈대가 거의 똑같이 반복된다:
C, 커널 내부, PCIe, DMA, 인터럽트, 디바이스 트리, 디버깅(gdb/ftrace). 그중에서도
Tesla 공고에 콕 집혀있던 문구 하나가 눈에 띄었다 — **"pre-silicon emulation"**. 칩이
실리콘으로 나오기 전에, 에뮬레이터 위에서 먼저 드라이버를 개발하는 방식이다. QEMU가
정확히 그 역할을 한다.

그래서 실제 하드웨어 없이, QEMU 위에서 가상 PCIe 장치를 만들고 그 드라이버를 처음부터
짜보기로 했다. 출발점은 QEMU가 교육용으로 내장하고 있는 `edu` 장치(`hw/misc/edu.c`) —
Masaryk 대학 커널 강의에서 쓰던 물건이다.

## 환경

- 호스트: WSL2 Ubuntu 24.04
- QEMU: 소스에서 직접 빌드(v9.1.0) — 이유는 간단하다, Phase 4에서 이 소스를 포크해서
  내 장치를 만들 거라 처음부터 소스 트리가 필요했다.
- 게스트: Ubuntu 24.04 cloud image + cloud-init, 9p로 호스트 레포를 게스트에 직접 마운트
  (호스트에서 코드 고치면 게스트에서 바로 `make`)

`lspci`로 확인:
```
00:04.0 Unclassified device [00ff]: Device 1234:11e8 (rev 10)
```

## BAR부터 — 이게 왜 MMIO인가

`lspci -vvv`로 더 들여다보면:
```
Region 0: Memory at fea00000 (32-bit, non-prefetchable) [size=1M]
```

여기서 세 가지를 짚을 만하다.

**Memory, I/O가 아니라**: x86 레거시 I/O 포트 공간은 통틀어 64KB뿐이다. edu의 레지스터
공간은 1MB — 애초에 I/O 포트 공간에 다 들어갈 수도 없다. 그러니 MMIO(Memory-Mapped I/O)
일 수밖에 없다.

**32-bit로 충분**: 이 장치가 필요로 하는 공간은 1MB뿐이라, 굳이 4GB 위쪽 주소를 쓸
이유가 없다.

**non-prefetchable인 이유**: edu의 레지스터 중엔 `0x04`(liveness check)처럼 "쓴 값을
반전시켜 저장하는" 부작용이 있는 레지스터가 있다. 이런 레지스터를 캐시하거나 미리
읽어버리면(prefetch) 그 부작용을 놓친다 — 그래서 non-prefetchable로 표시된다.

## 인터럽트가 실제로 올라가는 경로

edu의 실제 소스를 읽어보면, 인터럽트를 올리는 함수는 이렇게 생겼다:

```c
static void edu_raise_irq(EduState *edu, uint32_t val)
{
    edu->irq_status |= val;
    if (edu->irq_status) {
        if (edu_msi_enabled(edu)) {
            msi_notify(&edu->pdev, 0);
        } else {
            pci_set_irq(&edu->pdev, 1);
        }
    }
}
```

핵심은 **INTx와 MSI가 완전히 다른 종류의 신호**라는 것이다. INTx는 물리 핀 하나가
켜진 채로 유지되는 **레벨(level) 신호**라, 여러 장치가 같은 핀을 공유할 수 있는 대신
"이 인터럽트가 진짜 내 것인지" 드라이버가 직접 확인(상태 레지스터 읽기)하고 명시적으로
꺼줘야(ack) 한다. 반면 MSI는 특정 주소에 한 번 write하는 **일회성 메시지**라 핀 공유
문제 자체가 없다.

이 구분이 왜 중요한지는 Phase 2에서 뼈저리게 확인하게 된다 — ack를 빼먹었더니
인터럽트가 무한 폭주했다(2편 참고).

## DMA 레지스터 — 여기도 미리 봐두면 좋을 것

```
0x80 DMA source address
0x88 DMA destination address
0x90 DMA transfer count
0x98 DMA command (0x1 시작, 0x2 방향, 0x4 완료 시 인터럽트)
```

재밌는 건 DMA 완료 인터럽트도 결국 같은 `edu_raise_irq`를 그대로 재사용한다는 것 —
"팩토리얼 계산 끝남"과 "DMA 끝남"은 서로 다른 이벤트지만, 인터럽트를 올리는 매커니즘
자체는 하나로 통일돼 있다.

다음 편에서는 이 스펙을 바탕으로 실제 첫 PCI 드라이버(probe/BAR 매핑/레지스터 read)를
짠다.
