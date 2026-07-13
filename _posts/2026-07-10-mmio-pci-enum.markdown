---
layout: post
title: "QEMU로 PCIe 가속기 드라이버 바닥부터 ② — MMIO와 PCI 열거"
date: 2026-07-10 09:00:00 +0900
categories: driver linux-kernel pci
description: "PCI 드라이버의 probe/remove 뼈대, drvdata로 장치 인스턴스별 상태 관리, goto 체인으로 자원 정리하는 커널 관용구를 다룬다."
---

## 뼈대

리눅스 PCI 드라이버는 커널이 요구하는 정해진 형태가 있다:

```c
static const struct pci_device_id edu_ids[] = {
    { PCI_DEVICE(0x1234, 0x11e8) },
    { 0, }
};
MODULE_DEVICE_TABLE(pci, edu_ids);

static struct pci_driver edu_driver = {
    .name     = "edu",
    .id_table = edu_ids,
    .probe    = edu_probe,
    .remove   = edu_remove,
};
module_pci_driver(edu_driver);
```

`id_table`은 두 군데서 쓰인다 — 커널 PCI 코어는 이미 로드된 드라이버가 새로 나타난
장치와 매칭될 때 이걸 보고, `MODULE_DEVICE_TABLE`은 이 정보를 `.ko` 파일의 메타데이터로
박아넣어서 **모듈이 로드되기도 전에** `udev`/`modprobe`가 "이 장치엔 이 모듈이 필요하다"고
판단할 수 있게 해준다. 즉 하나는 "이미 뜬 드라이버가 장치를 인식"하는 용도, 하나는
"애초에 어떤 모듈을 띄울지 결정"하는 용도 — 완전히 다른 시점에 쓰인다.

## `pci_enable_device`만으로는 부족하다

`probe()`가 호출됐다고 장치가 바로 응답 가능한 상태는 아니다. PCI Command 레지스터의
"Memory Space Enable" 비트가 꺼져 있으면, BAR가 이미 주소를 배정받았어도 그 주소로
접근하면 아무도 응답을 안 한다(관례상 `0xFFFFFFFF`가 돌아온다). `pci_enable_device()`가
이 비트를 켜준다.

```c
ret = pci_enable_device(pdev);
if (ret) return ret;

regs = pci_iomap(pdev, 0, 0);   // 물리주소(BAR) -> 커널 가상주소
value = ioread32(regs + 0x00);  // identification 레지스터
```

첫 시도에서 `pci_iomap`의 리턴 타입(`void __iomem *`)을 `int`에 담았다가 크래시를 봤다 —
64비트 포인터를 32비트 정수에 담으면 주소가 잘려서 완전히 엉뚱한 곳을 가리키게 된다.
`void __iomem *` 그대로 받아야 한다.

## 장치 인스턴스별 상태 — drvdata

`probe`에서 확보한 정보(매핑된 주소 등)를 나중에 `remove`나 인터럽트 핸들러에서도
써야 하는데, 지역변수로 두면 함수가 끝나는 순간 사라진다. 그렇다고 전역변수로 두면
장치가 2개 이상 꽂혔을 때 서로 덮어써버린다.

해법은 리눅스 드라이버 코어가 제공하는 **drvdata** 패턴이다:

```c
struct edu_dev {
    void __iomem *regs;
    int irq_num;
};

edu = kzalloc(sizeof(*edu), GFP_KERNEL);
edu->regs = pci_iomap(pdev, 0, 0);
pci_set_drvdata(pdev, edu);   // 이 장치 인스턴스에 매달아둠

// 나중에, 다른 함수에서
struct edu_dev *edu = pci_get_drvdata(pdev);
```

이 패턴은 이후 인터럽트 핸들러의 `dev_id`, 문자 디바이스의 `filp->private_data`에서도
계속 반복해서 쓰이게 된다 — "이 함수 호출 체인 밖으로 상태를 들고 나가야 할 때"의
표준 답이다.

## 자원 정리는 역순으로, `goto` 체인으로

`probe()` 안에서 여러 단계(enable, kzalloc, iomap, irq, DMA...)를 거치는데, 중간
어딘가에서 실패하면 그때까지 확보한 것만 정확히 역순으로 풀어줘야 한다. C에 이걸 위한
관용구가 있다:

```c
err_unmap:
    pci_iounmap(pdev, edu->regs);
err_free:
    kfree(edu);
err_disable:
    pci_disable_device(pdev);
    return ret;
```

`goto`가 응용 코드에서는 기피 대상이지만, 커널 코딩 스타일에서는 "Centralized exiting
of functions"라는 이름으로 공식 권장되는 패턴이다. 실패 시 라벨이 순서대로 흘러
내려가면서(fallthrough) 그때까지 확보한 자원만 정확히 해제한다.

다음 편은 인터럽트 — 그리고 ack를 깜빡했을 때 실제로 무슨 일이 일어나는지.
