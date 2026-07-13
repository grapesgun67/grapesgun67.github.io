---
layout: post
title: "QEMU로 PCIe 가속기 드라이버 바닥부터 ③ — 인터럽트, 그리고 ack를 깜빡했을 때"
date: 2026-07-11 09:00:00 +0900
categories: driver linux-kernel interrupts
description: "공유 IRQ 핸들러 작성, INTx 레벨 신호에서 ack를 빼먹으면 실제로 벌어지는 인터럽트 스톰, 폴링과 completion의 차이."
---

## 벡터 확보 → 핸들러 등록

```c
nr = pci_alloc_irq_vectors(pdev, 1, 1, PCI_IRQ_LEGACY | PCI_IRQ_MSI);
edu->irq_num = pci_irq_vector(pdev, 0);
request_irq(edu->irq_num, edu_irq_handler, IRQF_SHARED, "edu", edu);
```

`pci_alloc_irq_vectors`가 MSI를 쓸지 레거시 INTx를 쓸지는 커널이 알아서 고른다 —
드라이버는 "어느 쪽이든 되는 대로"라고만 요청하고, 실제로 뭐가 잡혔는지는
`cat /proc/interrupts`로 확인해야 안다. 내 환경에서는 `IO-APIC`(레거시 INTx)로 잡혔다.

## 핸들러의 뼈대 — 공유 인터럽트를 전제로

```c
irqreturn_t edu_irq_handler(int irq, void *dev_id)
{
    struct edu_dev *edu = dev_id;
    u32 val = ioread32(edu->regs + REG_INTERRUPT_STATUS);

    if (val & FACT_IRQ) {
        iowrite32(val, edu->regs + REG_INTERRUPT_ACK);
        return IRQ_HANDLED;
    }
    return IRQ_NONE;
}
```

INTx는 핀 하나를 여러 장치가 공유할 수 있다. IRQ 11이 울리면 커널은 그 라인에 등록된
핸들러를 **전부** 호출하고, 각 핸들러는 자기 상태 레지스터를 봐서 "이거 내 것 맞나"를
스스로 판단한다. 내 것이 아니면 `IRQ_NONE`을 돌려줘야 커널이 다음 핸들러로 넘어간다.

## Ack를 빼먹으면 — 실제로 재현해본 인터럽트 스톰

처음엔 위 핸들러에서 ack 줄을 빼고 테스트했다. 결과는 dmesg가 같은 로그로 도배되는
인터럽트 폭주였다.

이유는 INTx가 **레벨(level) 신호**이기 때문이다. `edu_raise_irq`가 부르는
`pci_set_irq(&edu->pdev, 1)`은 "핀을 켜라"가 아니라 "핀을 계속 켜진 상태로 유지해라"에
가깝다. 커널이 인터럽트를 처리하고 다시 받아들일 준비가 됐을 때, 핀이 여전히 켜져
있으면 IOAPIC은 즉시 다시 통보한다 — 새 이벤트가 생긴 게 아니라, **미해결 이벤트
하나가 계속 재통보되는 것**이다.

```c
static void edu_lower_irq(EduState *edu, uint32_t val)
{
    edu->irq_status &= ~val;
    if (!edu->irq_status && !edu_msi_enabled(edu))
        pci_set_irq(&edu->pdev, 0);   // irq_status가 완전히 0일 때만 핀을 내림
}
```

`0x64`(ack 레지스터)에 값을 쓰는 게 바로 이 `edu_lower_irq`를 호출하는 트리거다. 안
쓰면 `irq_status`가 절대 0이 안 되고, 핀은 영원히 켜진 채로 남는다.

## 완료를 "기다리는" 두 가지 방법 — 폴링 vs `completion`

가장 단순한 방법은 CPU가 레지스터를 계속 다시 읽는 폴링이다:
```c
while (ioread32(regs + REG_CMD) & 0x1)
    ;
```
동작은 하지만 CPU 사이클을 낭비한다. 인터럽트가 있는 이유 자체가 이걸 피하기 위해서다.
`struct completion`을 쓰면 프로세스를 진짜로 재워두고, 인터럽트 핸들러가 `complete()`를
불러줄 때만 깨어난다:

```c
init_completion(&edu->done);
iowrite32(0x01, regs + REG_CMD);
wait_for_completion(&edu->done);   // 여기서 잠듦, CPU 자유

// 핸들러 안
complete(&edu->done);              // 여기서 깨움
```

이건 완전히 수동 시스템이다 — `complete()`를 어딘가에서 반드시 불러줘야 하고, 안 부르면
`wait_for_completion`은 영원히 안 끝난다(ack 안 하면 폭주, complete 안 하면 무한 대기 —
정반대 방향의 실수지만 둘 다 "인터럽트 흐름을 완전히 손으로 관리해야 한다"는 같은
교훈으로 이어진다).

다음 편은 DMA — 그리고 QEMU가 DMA를 100ms 뒤에 비동기로 처리한다는 걸 몰라서 겪은 삽질.
