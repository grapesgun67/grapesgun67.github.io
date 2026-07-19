---
layout: post
title: "RPi5 ↔ Nucleo SPI DMA 드라이버 바닥부터 ② — 커스텀 캐릭터 드라이버와 GPIO 인터럽트"
date: 2026-07-19 09:10:00 +0900
categories: driver linux-kernel spi interrupt
description: "spidev를 걷어내고 GPIO data-ready 인터럽트 기반 SPI 캐릭터 드라이버로. DT override, vendor prefix 매칭, 그리고 코드도 배선도 멀쩡한데 계속 0만 나오던 며칠."
---

[지난 편]({% post_url 2026-07-19-lv2-bootchain-bringup %})에서 `spidev`로 raw 바이트를
왕복시키는 데까진 성공했다. 근데 이 방식은 **RPi5가 언제 물어봐야 할지 알 방법이 없다** —
그냥 아무 때나 클럭을 쳐서 그 순간의 스냅샷을 받아올 뿐이라, 요청과 요청 사이에 갱신된
값은 그냥 사라진다. 이번 편의 목표는 Nucleo가 "새 데이터 준비됐다"고 먼저 알려주는 이벤트
기반 구조로 바꾸는 것.

## 기존 노드를 재활용해야 했던 이유

처음엔 새 DT 노드를 추가하는 방식으로 시도했다가 부팅 자체가 깨졌다. `bcm2712-rpi.dtsi`를
까보니 `dtparam=spi=on`이 이미 `spidev@0`이라는 노드를 만들어두고 있었고, 같은 chip-select
자리에 노드를 또 추가하니 병합이 깨졌던 것. 해결은 새 노드 추가가 아니라 **기존
`spidev0` 라벨을 override**하는 방식 — `compatible`만 우리 걸로 바꾸고
`data-ready-gpios` 프로퍼티를 추가했다.

## "no spi_device_id" 경고와 vendor prefix

컴파일 자체는 됐는데 커널이 "no spi_device_id" 경고를 냈다. `drivers/spi/spi.c`의
`__spi_register_driver()`를 직접 열어보니, DT의 `compatible` 문자열(`kim,spi-telemetry`)에서
**콤마 앞의 벤더 프리픽스(`kim,`)를 잘라낸 나머지(`spi-telemetry`)**와 정확히 일치하는
이름을 드라이버의 `spi_device_id` 테이블에서 찾는다는 걸 확인했다. 이 테이블을 추가하니
경고가 사라졌다.

## 이벤트 기반 구조 — top half / bottom half

`devm_gpiod_get()`으로 DT에서 GPIO 디스크립터를 얻고 `gpiod_to_irq()`로 IRQ 번호로
변환한 뒤, `devm_request_threaded_irq()`로 등록했다. top half(하드 IRQ 핸들러)는 안
쓰고(`NULL`) bottom half만 지정 — 실제 SPI 트랜잭션(`spi_sync()`)이 블로킹 호출이라
스레드 컨텍스트에서만 안전하게 부를 수 있기 때문이다. bottom half가 데이터를 읽어
버퍼에 저장하고 wait queue를 깨우면, 캐릭터 디바이스의 `read()`가
`wait_event_interruptible()`로 그 값을 기다렸다가 유저스페이스로 복사한다.

## 며칠을 날린 "계속 0만 나옴" 미스터리

드라이버를 다 완성했는데 `read()`가 계속 0만 반환했다. 소거법으로 하나씩 좁혀나갔다:

1. `spi_read()`(암묵적 tx_buf=NULL) 의심 → 명시적 `spi_sync()` 직접 구성 → 여전히 0
2. SPI 클럭 속도 의심 → 낮춰서 재시도 → 여전히 0 (나중에 보니 `od -tu4`로 확인하던 값
   자체가 엔디안을 잘못 읽고 있었던 것 — 이 가설은 애초에 틀렸었다)
3. 우리 드라이버를 완전히 배제 — **지난 편에서 확실히 됐던 raw spidev+파이썬 테스트를
   지금 배선 그대로 재실행** → **이것도 0**. 코드 문제가 전혀 아니라는 결론
4. 배선 육안 재확인 → 이상 없음
5. **Nucleo+RPi5 완전 전원 재순환** → 정상 작동

코드도 배선도 처음부터 멀쩡했다. 반복된 재플래시·재부팅 와중에 STM32의 SPI2 페리페럴이
소프트 리셋으로는 안 풀리는 상태로 꼬였던 것으로 보인다.

**교훈**: 소프트웨어적으로 설명 가능한 원인부터 순서대로 배제해나가되, 전부 배제되면
하드웨어 상태 자체(전원 재순환)도 고려해야 한다.

## teach-back

> "phase2에서는 python3스크립트가 xfer를 하면 spi 응답이 오는 형태. phase3는 nucleo가
> 데이터 준비되면 gpio로 인터럽트를 보내고, 커널드라이버가 인터럽트를 수신해서
> 인터럽트핸들러로 데이터를 갱신하는 등 처리를한다. 즉 인터럽트 기반으로 동작하므로
> 이벤트 기반이라 부를 수있다."

SPI 버스를 실제로 구동하는 주체(RPi5=마스터)는 이전 편 내내 안 바뀐다 — 바뀐 건 "RPi5가
언제 클럭을 치기로 결정하는가"의 근거다. 임의의 스크립트 호출 시점에서, Nucleo가 알려주는
정확한 갱신 시점으로.

다음 편은 이 SPI 전송이 실제로 PIO로 도는지 DMA로 도는지 확인하는 이야기 — 그리고
"DMA 채널이 DT에 있다"와 "이 전송이 실제로 DMA를 탄다"가 왜 별개의 질문인지.
