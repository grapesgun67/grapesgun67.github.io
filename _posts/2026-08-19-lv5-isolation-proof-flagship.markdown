---
layout: post
title: "RPi5 하이퍼바이저 격리 드라이버 바닥부터 ⑥(대표실험) — 인포테인먼트가 무너져도 사는 경로"
date: 2026-08-19 09:00:00 +0900
categories: driver linux-kernel virtualization isolation automotive jitter
series: 6
description: "Host를 stress-ng로 폭주시켰을 때, 격리 없이는 지터가 3.3배로 튀고 격리를 걸면 1.5배로 억제된다 — Lv5 전체를 관통하는 대표 실험의 실측 결과. 완전한 flat은 아니지만, 그래서 더 정직한 숫자."
---

여기까지 다섯 편에 걸쳐 쌓아온 것들 — [EL/TrustZone 이론]({% post_url
2026-08-10-lv5-arm-architecture-basics %}), [코어 격리의 한계]({% post_url
2026-08-11-lv5-rt-baseline-honest-conclusion %}), [KVM 인프라]({% post_url
2026-08-12-lv5-kvm-showcase-tcg-vs-kvm %}), [격리 도메인 결정]({% post_url
2026-08-12-lv5-isolation-domain-decision %}), [zero-copy 파이프라인]({% post_url
2026-08-18-lv5-ivshmem-zero-copy-pipeline %}) — 이 전부가 이 한 실험을 위한 준비였다.
Lv5의 포트폴리오 한 줄 요약이 정확히 이거다: **"인포테인먼트(Host)가 죽어도 안전
기능(격리된 Guest의 텔레메트리 경로)은 산다."** 지금까지는 이 문장이 설계 의도였다면,
이번 편은 이걸 숫자로 증명한다.

## 측정 설계 — 2×2, 조건당 10 trial

**격리(isolation) 무/유 × 부하(load) 무/유**, 4조건. 각 조건 10회 반복, trial당
`guest_consumer` 300샘플(~1초 분량).

- **격리 유** = [P3에서 검증된]({% post_url 2026-08-12-lv5-isolation-domain-decision %})
  그 설정 그대로 — QEMU vCPU 스레드에 `taskset -pc <격리코어>` + `chrt -f -p 90`. 격리
  무 = 아무것도 안 건 기본 상태(SCHED_OTHER, 코어 고정 없음).
- **부하 유** = 격리 코어를 제외한 나머지 코어에 `stress-ng --cpu --vm`([P1/P3와 동일
  방식·강도]({% post_url 2026-08-11-lv5-rt-baseline-honest-conclusion %}) 재사용 —
  비교 가능성을 유지하려고).
- `isolcpus`/`nohz_full` 같은 부팅타임 설정은 P1에서 이미 "이 워크로드엔 유의미한 효과가
  없다"고 결론난 변수라 조건 축에서 뺐다 — 이번엔 P3가 실제로 효과를 낸 vCPU 스레드 RT
  승격/핀닝만 다룬다.
- **지표**: `guest_consumer`가 P4에서부터 찍고 있던 두 값을 그대로 쓴다. **inter-arrival**
  (게스트 단일 클럭 — 신뢰 가능, 1차 지표)과 **latency**(Host-Guest 클럭 비교, P4에서
  이미 wall-clock 오프셋으로 오염됨을 확인했으니 절대값은 버리고 조건 간 jitter 폭만
  참고용).

## 자동화 — 재부팅 없이 4조건 전부

`lv5/p5-isolation-proof/measure.sh`가 호스트 쪽을 전부 자동화한다. 격리는 vCPU 스레드에
런타임으로 걸었다 빼는 것뿐이라 [P1처럼]({% post_url 2026-08-11-lv5-rt-baseline-honest-conclusion
%}) 조건마다 재부팅할 필요가 없다 — QEMU가 `-nographic`이라 게스트 콘솔이 곧 호스트
프로세스의 stdio라는 점을 이용해서, tmux 세션 안에 QEMU를 띄워두고 `tmux send-keys`/
`capture-pane`으로 `guest_consumer` 명령을 주입하고 출력을 캡처해 CSV로 쌓는다.

측정 자동화 중 실제로 겪은 문제가 둘 있었다.

**하나, tmux 소켓 불일치.** `measure.sh`를 `sudo`로 돌리는데 tmux 세션은 일반 계정으로
만들어서 세션을 못 찾았다 — tmux는 계정별로 소켓이 다르다(`/tmp/tmux-<uid>/...`).
`sudo tmux new -s p5guest`로 처음부터 root 소켓에 세션을 만들도록 고쳤다.

**둘, vCPU 스레드 이름이 안 붙는 QEMU 빌드.** vCPU 스레드를 이름으로 찾으려던 1차
구현(`ps -T` + `grep -iE 'CPU|kvm-vcpu'`)이 실패했다 — 이 QEMU 빌드에서는 스레드
`comm`이 전부 `qemu-system-aar`로 동일했다(이름이 안 붙는 빌드). `set -e`+`pipefail`
때문에 grep 매치 실패가 에러 메시지도 없이 스크립트를 통째로 죽여서, 처음엔 원인
파악이 [P1에서 겪었던 것과 같은 부류로]({% post_url
2026-08-11-lv5-rt-baseline-honest-conclusion %}) 까다로웠다. 고친 방법은 QEMU를
`-monitor tcp:127.0.0.1:4444,server,nowait`로 띄우고, bash 내장 `/dev/tcp` 가상
디바이스로 그 포트에 직접 접속해서 HMP `info cpus` 명령의 `thread_id=` 값을 파싱하는
것 — 소켓 도구(`socat`/`nc`) 설치 없이 QEMU 스스로 보고하는 값을 직접 받아오는 방식으로
바꿨다. 이름으로 찾는 게 아니라 하이퍼바이저한테 직접 물어보는 쪽이 훨씬 견고했다.

## 결과

**inter-arrival jitter(max−min, 1차 지표, us)**:

| 조건 | n | 평균 주기 평균±sd | 지터 평균±sd |
|---|---|---|---|
| 격리 없음 / 무부하 | 10 | 3460±3 | 2587±454 |
| 격리 없음 / 부하 | 10 | 3533±57 | **8424±2398** |
| 격리 / 무부하 | 10 | 3460±4 | 2642±405 |
| 격리 / 부하 | 10 | 3460±6 | **3925±1289** |

Host에 `stress-ng` 부하를 걸면, **격리 없이는 지터가 무부하 대비 3.3배로 튄다**
(2587us → 8424us). 같은 부하를 걸어도 **격리(vCPU `taskset`+`chrt -f -p 90`)를 걸어두면
증가폭이 1.5배로 줄어든다**(2587us → 3925us).

정직하게 짚을 부분: 완전히 flat한 건 아니다. 격리 상태에서도 부하를 걸면 지터는
여전히 늘어난다 — [P3에서 이미 확인했듯]({% post_url
2026-08-12-lv5-isolation-domain-decision %}) 이 칩엔 MPAM 같은 하드웨어 메모리 대역폭
파티셔닝이 없어서, 완벽한 무관함(Lv4 M4가 물리적으로 별도 버스를 써서 얻었던 그 수준)을
이 조합으론 낼 수 없다. 그래도 **격리 없을 때보다 확연히 덜 흔들린다**는 게 정직하고도
의미 있는 결론이다. 그리고 평균 주기(`ia avg`, ~3.46~3.53ms)는 4조건 모두 거의 동일했다
— 영향받는 건 평균이 아니라 **꼬리 변동폭(지터)뿐**이라는 것도 확인됐다.

참고용으로 본 latency jitter도 같은 방향(격리 없음/부하가 격리/부하보다 평균 높음)을
가리켰지만 표준편차가 커서 1차 지표만큼 깔끔하진 않았다 — inter-arrival을 1차 근거로
못박아둔 판단이 맞았던 셈이다.

## 왜 이 숫자가 의미 있는가

이 실험 하나로 Lv5 전체가 하나의 서사로 닫힌다. [P1]({% post_url
2026-08-11-lv5-rt-baseline-honest-conclusion %})에서 순수 코어 격리(isolcpus/PREEMPT_RT)
만으로는 이 워크로드의 지터를 통계적으로 못 줄인다는 걸 정직하게 확인했고, [P3]({% post_url
2026-08-12-lv5-isolation-domain-decision %})에서 KVM 게스트 경계 자체는 "제대로 세팅만
하면"(호스트 vCPU 스레드 RT 승격) 순수 코어 격리와 동등 이상이 될 수 있다는 걸
확인했다. 이번 P5는 그 "제대로 세팅한" 격리를 **실제 부하 조건에서 실제 텔레메트리
경로에** 적용해서, 처음 목표했던 "인포테인먼트가 죽어도 안전 기능은 산다"를 상대
비교로(과장 없이) 증명한다.

절대값을 "µs 단위로 보장한다"고 주장하지 않는다 — MPAM이 없는 이 하드웨어에서 그건
거짓말이 될 것이다. 대신 증명하는 건 "**같은 하드웨어, 같은 부하에서, 격리를 걸고 안
걸고의 차이가 이만큼 난다**"는 상대적이지만 재현 가능한 사실이다. 자동차 SDV 맥락으로
옮기면: IVI가 아무리 무거운 작업(내비게이션 렌더링, 미디어 재생, OTA 업데이트 처리)으로
CPU를 잡아먹어도, 격리된 도메인의 안전 관련 취득 경로는 "덜 흔들리는" 게 아니라
"확연히 덜 흔들린다"는 실측 근거를 갖게 된 것이다.

## 챕터를 닫으며

P0의 EL 이론부터 시작해서, P1의 정직한 실패, P2의 KVM 인프라, P3의 리스크 관문 통과,
P4의 zero-copy 파이프라인, 그리고 이번 P5의 대표 실험까지 — Lv4에서 물리적으로 분리된
버스(M4의 AHB)로 증명했던 것과 정확히 대칭을 이루는 방식으로, "같은 버스를 공유하는
가상화 환경에서도 격리를 제대로 설계하면 부하 영향을 확연히 줄일 수 있다"를 실물
RPi5에서 증명했다.

다음 편(챕터 마지막)은 TrustZone/OP-TEE로 방향을 튼다 — STM32MP157F-DK2 실물에서
서명 키를 secure storage에 보관하고 서명해주는 Trusted App을 자작하고, SMC 호출
경로를 ftrace로 직접 들여다본다.
