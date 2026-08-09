---
layout: post
title: "STM32MP157F-DK2로 넘어와서 ③ — ADC를 M4에 넘기고, 리눅스를 두들겨도 안 흔들리는지 실측하다"
date: 2026-08-07 09:00:00 +0900
categories: driver linux-kernel arm stm32mp1 mixed-criticality adc
series: 5
description: "Lv4 전체가 이 실험을 위한 인프라였다 — M4가 ADC를 자율 샘플링해서 리눅스로 밀어넣는 동안, 리눅스에 CPU/메모리 부하를 걸어도 M4의 샘플링 타이밍이 흔들리지 않는지 실측했다."
---

[2편]({% post_url 2026-08-05-lv4mp1-rpmsg-driver-latency %})에서 RPMsg 왕복 지연을
쟀다. Lv4의 진짜 목표는 지연 숫자 자체가 아니라 — "코어1(M4)이 실시간 작업을 하는 동안,
코어0(리눅스)에 아무리 부하를 걸어도 그 작업이 안 흔들린다"는 것. 이번 편이 그 증명이다.

## 주장의 경계부터 정확히 긋기

**"리눅스를 폭주시켜도 대시보드가 안 느려진다"는 주장이 아니다.** 대시보드 갱신(A7이
배치를 소비하는 경로)은 부하 중 실제로 느려지거나 버벅일 수 있고, 그게 정상이다. 측정을
두 갈래로 분리했다:

- **M4쪽 타임스탬프**(진짜 측정 대상): ADC 샘플링 순간 M4 자신의 타이머(`DWT->CYCCNT`,
  Cortex-M 내장 사이클 카운터)로 찍는다 — A7의 GIC/스케줄러와 무관한 M4 자체 NVIC
  인터럽트라 A7 부하와 무관하게 flat해야 한다.
- **A7쪽 타임스탬프**(대조군): 리눅스가 배치를 실제로 소비한 시각 — 부하 중 지연/지터가
  생기는 게 정상이고, 오히려 이 대비가 증명의 핵심이다.

"전부 다 빠르다"가 아니라 "뭐가 보호되고 뭐가 안 보호되는지 경계를 정확히 긋는" 게
정직한 실험 설계라고 판단했다.

## ADC 소유권을 M4로 넘기기

`ls /sys/bus/iio/devices/`로 확인해보니 ADC1/ADC2 둘 다 이미 리눅스 IIO 서브시스템이
점유 중이었다 — P0에서 코어1을 리눅스 스케줄링에서 떼어냈던 것과 같은 종류의 "리소스
소유권" 문제가 ADC에도 있었다. DT를 직접 패치해야 하나 걱정했는데, 커널 소스를 뒤져보니
`stm32mp157f-dk2-m4-examples.dts`가 이미 존재했다:

```
&adc  { status = "disabled"; };
&m4_adc  { vref-supply = <&vrefbuf>; status = "okay"; };
```

ADC/DAC/DMA2/타이머를 통째로 A7에서 떼어 `m4_*` 네임스페이스로 재선언해둔 DTB — 우리가
쓰려는 M4 예제(`ADC_SingleConversion_TriggerTimer_DMA`, ADC2 채널16+TIM2+DMA)가 쓰는
페리페럴 세트와 정확히 일치했다. 직접 DT 패치 없이 U-Boot `extlinux.conf`에 두 번째
`LABEL`만 추가해서 부팅 전환.

## M4 펌웨어 — 자율 전송, 그리고 새로운 함정

베이스를 2편의 `OpenAMP_TTY_echo_latency`에서 ST의 `OpenAMP_TTY_echo_wakeup` 예제로
바꿨다 — OpenAMP 인프라와 ADC2/TIM2/DMA2 고정주기 샘플링이 이미 결합된 조합이라
안전했다. 새 엔드포인트(`"rpmsg-virtadc"`)를 추가해서 `struct adc_batch_msg{seq,
delta_cycles, samples[32]}`를 DMA half/full 콜백에서 **리눅스 쪽 트리거 없이 자율
전송**하게 만들었다. 이게 2편의 `rpmsg-latency`(요청→응답)와는 근본적으로 다른 설계라
새로운 함정이 하나 있었다 — OpenAMP의 `ept->dest_addr`는 `RPMSG_ADDR_ANY`로 시작해서
**첫 수신 메시지**에서만 자동으로 채워지는데, M4가 리눅스로부터 아무것도 안 받는 이
구조에서는 그 채움이 영원히 안 일어난다. 커널 드라이버의 `probe()`에서 킥오프 메시지
하나를 먼저 보내는 것으로 해결했다.

## 대시보드 — pip도 stress-ng도 없는 이미지에서

`lv4-mp1/dashboard/`를 새로 배포하면서 WebSocket을 쓰려 했는데, 보드 이미지에 `pip`도
`opkg`도 없었다(최소 이미지, Yocto 재빌드 없이는 설치 불가). **SSE**(`http.server`
표준 라이브러리 + 브라우저 내장 `EventSource`)로 전환해서 추가 설치 없이 해결했다.
`stress-ng`도 마찬가지로 없어서, 순수 `multiprocessing`으로 부하 생성기(`loadgen.py`)를
직접 작성했다 — `cpu` 모드(코어 스핀), `vm` 모드(코어당 32MB 버퍼 `memcpy` 반복으로
실제 DRAM 트래픽 유발).

<div class="demo-images">
  <figure>
    <img src="{{ '/assets/videos/lv4mp1-dashboard-initial.png' | relative_url }}" alt="대시보드 초기 화면">
    <figcaption>실행 직후 — 배치가 아직 안 쌓인 초기 화면</figcaption>
  </figure>
  <figure>
    <img src="{{ '/assets/videos/lv4mp1-dashboard-midstream.png' | relative_url }}" alt="대시보드 스트리밍 중 화면">
    <figcaption>스트리밍 중 — ADC 파형 + 배치 간격(지터) 차트 + 드롭 카운터</figcaption>
  </figure>
</div>

## 결과 — 무부하/CPU부하는 통계적으로 완전 동일

조건당 30초, 약 1,000배치씩 측정:

| 조건 | 평균(사이클) | 표준편차 | 드롭 |
|---|---|---|---|
| 무부하 | 6,271,983.0 | 42.0 | 0 / 1000 |
| CPU 부하 | 6,271,983.0 | 42.0 | 0 / 1000 |
| 메모리 부하 | 6,271,983.0 | 41.9 | 1 / 999 |

무부하와 CPU 부하는 평균·표준편차까지 소수점 자리수 그대로 동일했다. 메모리 대역폭
부하에서만 999개 중 1개가 드롭됐는데, 그마저도 살아남은 배치들의 타이밍은 흔들리지
않았다 — 지터가 "번지는" 게 아니라 드롭이 이진법적으로(성공 아니면 유실) 일어난다는 뜻.

숫자로는 이렇지만, 실제로 화면에서 어떻게 보이는지가 더 직관적이다. 두 영상 다 12초로
길이를 맞춰뒀다 — CPU 부하를 거는 동안과 메모리 부하를 거는 동안 대시보드가 어떻게
다르게(혹은 다르지 않게) 반응하는지 비교해보면 된다.

<div class="demo-videos">
  <div class="demo-video-item">
    <video id="video-cpu-load" playsinline controls controlsList="nodownload"></video>
    <p class="demo-video-label">CPU 부하 중 대시보드 — 지터/드롭 변화 없음</p>
  </div>
  <div class="demo-video-item">
    <video id="video-vm-load" playsinline controls controlsList="nodownload"></video>
    <p class="demo-video-label">메모리 부하 중 대시보드 — 드롭 카운터만 아주 가끔 1 증가</p>
  </div>
</div>
<button id="play-both-btn" class="demo-play-btn" type="button">▶ 같이 재생</button>

<script>
(function () {
  var v1 = document.getElementById('video-cpu-load');
  var v2 = document.getElementById('video-vm-load');
  var btn = document.getElementById('play-both-btn');
  var videos = [v1, v2];
  var syncing = false;

  v1.src = '{{ "/assets/videos/cpu-load-demo.mp4" | relative_url }}';
  v2.src = '{{ "/assets/videos/vm-load-demo.mp4" | relative_url }}';

  function playBoth() {
    videos.forEach(function (v) { v.currentTime = 0; });
    Promise.all(videos.map(function (v) { return v.play(); })).catch(function () {});
  }

  btn.addEventListener('click', playBoth);

  videos.forEach(function (v) {
    var other = v === v1 ? v2 : v1;

    v.addEventListener('play', function () {
      if (syncing || !other.paused) return;
      syncing = true;
      other.play().catch(function () {}).finally(function () { syncing = false; });
    });

    v.addEventListener('pause', function () {
      if (syncing || v.ended || other.paused) return;
      syncing = true;
      other.pause();
      syncing = false;
    });
  });
})();
</script>

## 왜 이런 결과가 나왔는가 — 처음 세운 가설을 스스로 정정하다

처음엔 "공유메모리/버스 경합에도 잘 버틴다"고 설명했는데, M4 링커스크립트를 다시
확인하니 부정확했다. `m_ipc_shm`(RPMsg vring이 있는 곳)은 `0x10040000`대인데, ST
공식 문서를 찾아보니 이 대역 전체가 외부 DDR이 아니라 **칩 내장 MCUSRAM**이고, 그것도
DDR이 물린 AXI가 아니라 **별도의 AHB 버스**에 있었다. `stress-ng`류 부하가 두들기는
자원(DDR/AXI)과 M4가 RPMsg에 쓰는 자원(AHB SRAM)이 애초에 물리적으로 다른 버스라 —
"경합에도 잘 버틴다"가 아니라 **"경합할 공유 자원 자체가 없다"**가 정확한 설명이었다.

그럼 드롭 1건은? 버스 경합이 아니라 A7쪽 소프트웨어 지연으로 설명하는 게 맞다. vring
슬롯 수는 16개(고정)뿐인데, A7이 `--vm` 부하로 캐시가 심하게 스래싱당하면 IPCC
인터럽트/vring 드레인 처리가 살짝 늦어질 수 있고, 그 사이 M4가 계속 새 배치를 채우다
슬롯이 순간적으로 꽉 차서 1건이 못 들어갔다는 게 훨씬 앞뒤가 맞는다.

**최종 경계선**: M4의 샘플링 타이밍 자체는 어떤 강도의 A7 부하에도 무조건 안전(물리적으로
다른 버스), 반면 리눅스로의 배달(vring 드레인)은 A7이 얼마나 바쁜지에 실제로 의존하고
메모리 압박이 심하면 아주 낮은 확률(0.1%)로 드롭 가능하다 — 이게 과장 없는 정직한
결론이다.

다음 편은 지금까지 손으로 크로스빌드하던 커널 모듈을 정식 Yocto 레이어로 패키징하고,
Zynq(챕터 4)와 이번 STM32MP1 두 접근을 정리해서 Lv4를 마무리하는 이야기다.
