---
layout: post
title: "RPi5 하이퍼바이저 격리 드라이버 바닥부터 ① — EL0~3와 TrustZone, VHE가 뒤집은 첫 가정"
date: 2026-08-10 09:00:00 +0900
categories: driver linux-kernel arm architecture hypervisor kvm trustzone
series: 6
description: "Lv5의 목표는 RPi5 한 칩 안에서 Host/Guest를 KVM으로 격리하고 TrustZone까지 붙이는 것. 시작은 EL0~3 이론인데, 'KVM이 모듈로 나중에 얹힌다'는 첫 가정이 dmesg 한 줄로 뒤집혔다."
---

Lv4까지는 한 칩 안에서 두 개의 **코어**(A7/M4)를 나누는 이야기였다. Lv5는 한 단계 더
간다 — RPi5 하나에서 Host(인포테인먼트 역할 Linux)와 Guest(Safety 역할 RT Linux)를 KVM으로
격리하고, 텔레메트리 취득 경로를 그 격리 도메인에 배치해서 "인포테인먼트가 죽어도 안전
기능은 산다"를 실증하는 게 목표다. 여기에 TrustZone/OP-TEE로 보안 서비스까지 얹는다.

당연히 시작은 이론이다 — EL(Exception Level), Stage 1/2 주소변환, PSCI, TF-A 부트체인.
근데 이 이론 정리 도중에 첫 가정부터 실측 한 줄에 뒤집혔다.

## EL0~3 — 특권 사다리를 하드웨어가 강제한다

32비트 시절(ARMv4~v7)엔 User/FIQ/IRQ/Supervisor/Abort 같은 "모드"들이 파편적으로 있었고,
TrustZone용 Monitor 모드나 가상화용 Hyp 모드는 한참 뒤에 땜질식으로 추가됐다. ARMv8-A가
이걸 EL0~EL3라는 일관된 계층으로 재설계했다 — 숫자가 높을수록 특권이 세다.

| EL | 이름 | 담당 |
|---|---|---|
| EL0 | Unprivileged | 유저 앱 |
| EL1 | OS Kernel | 커널 본체, 자기 프로세스의 Stage 1 관리 |
| EL2 | Hypervisor | 게스트의 Stage 2(IPA→PA) 관리, 게스트 민감동작 트랩 |
| EL3 | Secure Monitor | Secure/Non-secure 월드 전환의 유일한 관문 |

중요한 건 이게 소프트웨어 컨벤션이 아니라 **하드웨어가 강제**한다는 점이다. `MRS`/`MSR`로
시스템 레지스터를 건드릴 때마다 CPU가 디코드 단계에서 `CurrentEL`과 그 레지스터의 "최소
접근 EL"을 비교하고, 위반하면 그 자리에서 Undefined Instruction 예외를 던진다. 레벨 이동도
`SVC`(EL0→EL1)/`HVC`(EL1→EL2)/`SMC`(EL1·EL2→EL3) 전용 명령어로만 가능 — syscall 패턴이
층마다 반복되는 구조다.

(참고로 이 계층은 Cortex-A 전용이다. Lv4에서 썼던 M4(Cortex-M)는 Thread/Handler 모드뿐,
EL 개념 자체가 없다 — 멀티태스킹 OS나 가상화가 필요 없는 마이크로컨트롤러엔 애초에 불필요한
기계장치라서.)

## dmesg 한 줄에 뒤집힌 첫 가정

RPi5에서 실제로 확인해봤다:

```
$ dmesg | grep -i kvm
[    0.292112] kvm [1]: nv: 568 coarse grained trap handlers
[    0.292274] kvm [1]: IPA Size Limit: 40 bits
[    0.297395] kvm [1]: GICV region size/alignment is unsafe, using trapping (reduced performance)
[    0.303960] kvm [1]: vgic interrupt IRQ9
[    0.314252] kvm [1]: VHE mode initialized successfully
$ ls -l /dev/kvm
crw-rw---- 1 root kvm 10, 232 Jul 26 03:33 /dev/kvm
```

처음 가정은 "hyp stub만 세워두고 본 커널은 EL1로 내려가고, KVM 코드는 나중에 modprobe로
얹힌다"였다. **틀렸다.** 타임스탬프가 부팅 0.29초 시점이라는 것부터 KVM이 커널에 이미
컴파일되어 포함된 채로 부팅과 동시에 초기화된다는 뜻이고, 결정적으로 `VHE mode initialized
successfully` 한 줄이 그림 전체를 바꾼다.

VHE(Virtualization Host Extensions, ARMv8.1+)는 `HCR_EL2.E2H` 비트로 EL2의 주소변환
레지스터 구조를 EL1과 똑같은 모양으로 바꿔서, **호스트 커널 본체(스케줄러·드라이버 전부)가
코드 수정 없이 EL2에서 직접 상주**하게 해주는 기능이다. 즉 지금 이 순간 RPi5의 EL2는
절대 비어있지 않다 — VM이 하나도 안 떠 있어도 호스트 자체가 이미 EL2 거주자다.

여기서 한 번 더 헷갈렸던 지점: "호스트가 EL2에 산다"(VHE)와 "Stage 2 변환이 활성화되어
있다"는 서로 다른 축이라는 것. 지금은 게스트가 없으니 `VTTBR_EL2`엔 아무것도 안 걸려있고
Stage 2는 비활성 상태다 — 호스트는 EL2에서 돌지만 자기 Stage 1만 쓴다. Stage 2는 P2에서
kvmtool로 실제 게스트를 띄우는 순간 처음 켜진다.

## VA→IPA→PA, 그리고 "유저=EL1/커널=EL2"라는 오해

게스트 OS(EL1)는 자기 페이지테이블(Stage 1)로 VA를 변환한 결과를 진짜 물리주소라고
믿지만, 실제로는 하이퍼바이저(EL2)가 그 위에 한 겹 더 씌워놓는다 — 그 중간 결과가
**IPA(Intermediate Physical Address)**다. IPA→진짜 PA가 **Stage 2**(`VTTBR_EL2`, 호스트만
관리, 게스트는 존재 자체를 모름). dmesg의 `IPA Size Limit: 40 bits`가 이 얘기다.

처음엔 "유저공간=EL1이 관리, 커널공간=EL2가 관리"로 생각했는데 이것도 틀렸다. 정확한
그림은:

```
게스트 유저 VA ─┐
                ├─ Stage1(TTBR0_EL1, 게스트 관리) → IPA ─┐
게스트 커널 VA ─┘                                          ├─ Stage2(VTTBR_EL2, 호스트 관리) → 진짜 PA
                Stage1(TTBR1_EL1, 게스트 관리) → IPA ─────┘
```

`TTBR0_EL1`(유저)과 `TTBR1_EL1`(커널) **둘 다 EL1 소속**이다. 유저/커널 구분은 순전히
Stage 1 내부의 축이고 EL1 혼자 다 처리한다. `VTTBR_EL2`는 완전히 다른 축 — 유저든 커널이든
게스트가 만든 IPA는 예외 없이 이 두 번째 층을 거쳐야 진짜 PA가 나온다. 게스트가 뜬 상태의
메모리 접근 1회는 이론상 Stage 1(최대 4단계)+Stage 2(최대 4단계)를 다 순회한다(TLB가 캐싱해
매번 그러진 않지만).

## PSCI — Zynq에는 없었던 그 편의

PSCI(Power State Coordination Interface)는 코어 켜기/끄기/서스펜드/리셋을 ARM이 표준화한
SMC 기반 API다. Non-secure 쪽(Linux)은 표준 함수 ID로 SMC만 날리면 되고, 칩별 레지스터
조작은 EL3에 상주하는 펌웨어(BL31)가 대신 처리한다.

이게 Lv4 챕터랑 바로 연결되는 지점이다. **Zynq-7000엔 PSCI를 구현한 EL3 펌웨어가 아예
없었다** — 그래서 [Zybo 코어1을 깨울 때]({% post_url 2026-07-29-lv4-zynq-manual-bringup %})
mainline Linux가 표준 `psci_ops.cpu_on()` 대신 벤더 전용 `zynq_cpun_start()`를 직접
구현해서, SMC/EL3를 거치지 않고 EL1 커널 코드가 SLCR 레지스터(`A9_CPU_RST_CTRL`)를
직접 두드린다. 그때 U-Boot 콘솔에서 `mw.l`로 직접 했던 그 작업이, PSCI가 있었다면
펌웨어 뒤에 숨겨졌을 부분이었다. **PSCI 유무가 "코어 기동 로직이 어디에 노출되는가"를
가른 실제 사례**를 두 칩을 오가며 직접 본 셈이다. RPi5/STM32MP1처럼 TF-A가 있는 최신
칩은 `psci_ops.cpu_on()` 한 줄로 끝난다.

## TF-A 부트 스테이지, 그리고 두 번째로 틀렸던 순서 예측

| 스테이지 | 실행 위치 | 역할 |
|---|---|---|
| BL1 | 온칩 ROM(불변, root of trust), EL3 | 리셋 직후 최초 실행 |
| BL2 | EL3 | DRAM 초기화 + BL31/32/33 검증·로드 |
| BL31 | EL3, 부팅 후에도 영구 상주 | Secure Monitor 본체, PSCI+SMC 라우팅 |
| BL32(선택) | Secure EL1 | Secure World OS(OP-TEE) |
| BL33 | Non-secure | U-Boot → Linux |

ARMv8-A 코어는 리셋 시 하드웨어적으로 구현된 것 중 **가장 높은 EL에서 무조건** 실행을
시작한다 — 선택 사항이 아니다. 이유는 신뢰 사슬: 전원 직후 아무것도 검증 안 된 상태에서,
가장 신뢰되는(물리적으로 불변인 ROM) 코드가 가장 높은 특권으로 먼저 돌면서 다음 단계
서명을 검증해야만 위→아래로 신뢰가 성립한다.

여기서 두 번째로 틀렸다. "OP-TEE(BL32)가 U-Boot(BL33)보다 먼저 뜨는 건 높은 특권에서 낮은
특권 순으로 부팅되기 때문"이라고 추측했는데, EL 숫자만 보면 오히려 틀린 추측이다 —
OP-TEE는 Secure **EL1**, U-Boot는 보통 **EL2**라서 EL 사다리 기준으론 U-Boot가 더 높다.
실제 이유는 EL 번호가 아니라 **Secure/Non-secure World**라는 완전히 별도의 축이었다:
Non-secure 쪽이 SMC로 OP-TEE를 호출하려면 OP-TEE가 먼저 초기화되어 대기 중이어야 하고,
민감정보를 다루는 Secure World가 아직 신뢰 확립 전인 Non-secure OS보다 먼저 자리잡는
게 안전하기 때문이다. EL 사다리(각 월드 내부의 특권 단계)와 World(완전히 별도 축)를
하나로 뭉뚱그리면 이번처럼 순서 예측이 어긋난다.

부팅 순서 BL1→BL2→BL31→BL32→BL33→Linux는 이후 STM32MP157F-DK2에서 실측한 로그
(`TF-A BL2 → OP-TEE 4.0.0 → U-Boot 2023.10 → Linux 6.6.129`)와 정확히 일치했다 — BL1은
조용히 지나가고 BL2부터 로그에 찍힌다.

## RPi5는 진짜 TF-A를 쓰나? — 로그가 이상해서 찾아봤다

STM32MP1과 달리 RPi5 dmesg엔 "TF-A" 배너가 안 보이고 `raspberrypi-firmware`/
`rp1-firmware`(브로드컴 자체 펌웨어)만 보였다. `psci: PSCIv1.1 detected in firmware.`/
`SMC Calling Convention v1.2`는 찍히니 EL3 펌웨어가 PSCI를 구현 중인 건 확실한데, 구조가
궁금해서 찾아봤다.

- RPi 재단이 TF-A 공식 fork(`raspberrypi/arm-trusted-firmware`)를 유지하고 mainline
  TF-A에도 `plat/rpi5` 포트가 있다 — RPi5도 진짜 TF-A를 쓰지만, STM32MP1과 부트체인 구조가
  다르다.
- STM32MP1은 TF-A가 BL1/BL2/BL31을 전부 담당한다. **RPi5는 브로드컴 자체 ROM+GPU
  펌웨어가 BL1/BL2 역할을 대신하고, `config.txt`의 `armstub=bl31.bin`으로 TF-A의 BL31만
  로드해서 EL3로 점프한다** — TF-A가 모듈식이라 BL1/BL2 없이 BL31만 갖다 쓸 수 있다는 걸
  보여주는 실제 사례다.
- 공식 문서에 따르면 RPi5용 BL31은 "64비트 EL2 payload(Linux, EDK2)를 부팅하도록 만들어진
  최소 구현"이다 — STM32MP1처럼 BL33→EL1이 아니라 **BL31이 곧바로 EL2 상태로 페이로드를
  넘긴다.** 이게 RPi5에서 VHE-Linux가 자연스럽게 EL2에 상주하게 되는 구조적 이유이고,
  앞서 본 `VHE mode initialized successfully`가 여기서 나온다.
- 그리고 보안 관점의 중요한 caveat: 공식 문서는 "이 포트는 진짜 보안이 아니다 — DRAM에
  Secure 전용 메모리 컨트롤러 보호가 없어 Non-secure World에서도 접근 가능하다"고 명시한다.
  Lv5 계획이 처음부터 "RPi5는 시큐어월드 지원이 약해서 OP-TEE는 별도 보드(STM32MP157F-DK2)로
  승격한다"고 전제하고 있었는데, 그 근거가 실물 공식 문서로 확인된 셈이다.

## 다음

이론은 여기까지고, 다음 편은 실제로 `isolcpus`/`nohz_full`로 코어를 격리하고 cyclictest로
4분면을 실측한다 — 그리고 여기서도 정직한 결론이 하나 나온다: 예상만큼 깨끗하게 개선되지
않았다.
