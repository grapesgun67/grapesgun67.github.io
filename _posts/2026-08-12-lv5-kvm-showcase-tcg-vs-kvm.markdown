---
layout: post
title: "RPi5 하이퍼바이저 격리 드라이버 바닥부터 ③ — TCG보다 45배 빠른 이유를 손으로 확인하다"
date: 2026-08-12 09:00:00 +0900
categories: driver linux-kernel kvm kvmtool virtualization qemu
series: 6
description: "kvmtool로 최소 aarch64 게스트를 직접 빌드해 부팅하고, vCPU가 진짜 호스트 스레드라는 걸 taskset으로 확인한 다음, 같은 커널·같은 rootfs로 TCG와 KVM의 부팅 속도를 재봤다 — 45배 차이."
---

[2편]({% post_url 2026-08-11-lv5-rt-baseline-honest-conclusion %})의 정직한 결론(코어
격리만으론 안 된다) 뒤로, Lv5 P2는 방향을 바꿔 **KVM 자체를 손으로 만져본다.** 목표는
네 가지 — kvmtool로 최소 게스트 부팅, vCPU를 격리 코어에 고정, TCG(순수 소프트웨어
에뮬레이션) vs KVM(하드웨어 가속) 속도 비교, `perf kvm stat`로 VM exit 뜯어보기.

## VMM은 엔진이 아니라 두 조각이다

먼저 개념 하나만 짚고 가자 — **KVM 자체는 VM을 못 띄운다.** KVM은 리눅스 커널에 내장된
서브시스템으로, 게스트 명령어를 실제 CPU에서 돌게 하고 트랩을 받아 되돌려주는 **메커니즘**만
제공한다. "이 이미지를 이 주소에 올려", "이 게스트는 콘솔 하나만" 같은 정책과 실제 장치
흉내는 **유저스페이스**(kvmtool, QEMU)의 몫이다. 이렇게 나뉜 이유는 선택이 아니라 강제다 —
게스트 모드 진입/탈출은 CPU 특권 명령어라 하드웨어가 커널만 허용한다. 반면 "장치 흉내"는
특권이 필요 없는 일반 로직이라 유저스페이스에 두는 게 유리하다(버그 나도 프로세스만
죽지 커널은 안 죽는다).

`VMM = KVM(커널, 가속 엔진) + kvmtool/QEMU(유저스페이스, 정책+장치)`. QEMU는 TCG도
KVM도 되는 범용 VMM이라 코드가 방대한데, **kvmtool(=lkvm)은 KVM 전용으로만 만든 초경량
VMM**이다 — TCG 지원 자체가 없다. 코드가 작아서 "VMM이 실제로 뭘 하는지"가 소스만 읽어도
보인다는 게 이걸 고른 이유다.

## 게스트 커널 빌드 — 커널 상류가 이미 준비해둔 조각

호스트/RT 커널 빌드에 썼던 `third_party/rpi-linux`를 그대로 재사용했다. 호스트가 쓰는
`bcm2712_defconfig`를 손대는 대신, 커널 소스에 이미 있던 `arch/arm64/configs/virt.config`
조각을 발견해서 병합했다 — "VM 게스트로 쓸 때 불필요한 벤더 SoC 드라이버를 다 끄는" 용도로
커널 유지보수자들이 만들어둔 표준 워크플로였다.

```bash
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- defconfig
scripts/kconfig/merge_config.sh -m .config arch/arm64/configs/virt.config
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- olddefconfig
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- -j4 Image
```

rootfs는 [Lv4 Zynq 때]({% post_url 2026-07-29-lv4-zynq-manual-bringup %}) 만들어둔
busybox를 aarch64용으로 재빌드해서 재활용했다. 여기서 한 번 걸렸다 — 빌드(`make
CROSS_COMPILE=aarch64-linux-gnu-`)까지는 aarch64로 잘 됐는데, 그다음 `make
CONFIG_PREFIX=<경로> install`에서 `CROSS_COMPILE=`을 빼먹었다. busybox의 `install`
타겟이 `busybox` 타겟에 의존하고 있어서, make가 "다시 빌드해야 하나" 판단할 때
호스트 기본 gcc(x86-64)로 조용히 재컴파일해버린 것. 결과적으로 rootfs에 x86-64
바이너리가 심어졌고, aarch64 게스트 커널이 이걸 실행하려다 `ENOEXEC`로 커널 패닉
("No working init found")이 났다. `file` 명령으로 재확인하는 습관 덕분에 바로 잡았다.
**교훈: 크로스컴파일 프로젝트에서 `install`처럼 빌드를 유발할 수 있는 타겟은 매번
`CROSS_COMPILE=`을 명시해야 한다** — 한 번 정확한 툴체인으로 빌드했어도 후속 타겟에서
빠뜨리면 조용히 호스트 아키텍처로 되돌아갈 수 있다.

## 첫 부팅

```bash
sudo ./lkvm run -k Image -i rootfs.cpio.gz -m 256 -c 1 --console virtio \
  -p "console=hvc0 rdinit=/init"
```

virtio 드라이버 로드 → `/init` 실행 → 셸 진입까지 정상 동작. RPi5 위에 KVM 가속으로
최소 aarch64 게스트를 처음 성공적으로 띄운 순간이다.

## vCPU가 진짜 호스트 스레드라는 걸 확인하다 — 그리고 grep 하나에 속을 뻔했다

이론상 vCPU는 호스트 쪽 스레드 하나다. `ps -eLo pid,tid,comm,psr | grep -i lkvm`로 처음
확인했을 때 스레드가 딱 하나(`lkvm`, 메인 스레드)만 보여서 그걸 `taskset`으로 CPU3에
고정했는데 `psr`가 안 바뀌었다. **그릴 잘못 걸었던 것.**

`top -H -p <PID>`로 전체 스레드 목록을 다시 보니 kvmtool이 프로세스당 여러 스레드를 쓰고
있었다: `lkvm`(메인/모니터), `kvm-ipc`, `ioeventfd-worker`, `term-poll`,
`threadpool-worker`×4, 그리고 **`kvm-vcpu-0`**(진짜 vCPU 실행 스레드), `virtio-net-*`
(장치 에뮬레이션 워커들). `grep -i lkvm`이 스레드 이름에 "lkvm"이 안 들어간
`kvm-vcpu-0`을 걸러버려서 안 보였던 것이다. **멀티스레드 프로세스에서 특정 역할의
스레드를 찾을 땐 프로세스 이름이 아니라 `top -H`로 스레드 이름 전체 목록을 먼저 봐야
한다.**

정확한 TID(`kvm-vcpu-0`)에 `taskset -pc 3 <TID>` 적용 후, 게스트 안에서 카운터를 계속
돌리면서 `ps -eLo pid,tid,comm,psr`로 확인하니 `kvm-vcpu-0`의 `psr=3`이 고정돼 있었다.
**vCPU가 진짜 호스트 스레드라는 이론이 실전에서 그대로 확인됐다** — 평범한 `taskset`으로
게스트의 실행 코어를 그대로 고정시킬 수 있다는 뜻이다.

## TCG vs KVM — 45배

TCG 쪽은 `/dev/kvm`이 필요 없어서(크로스 아키텍처 소프트웨어 에뮬레이션이니까) WSL2에서
바로 실행 가능하다.

```bash
qemu-system-aarch64 -M virt -cpu max -accel tcg -m 256 -smp 1 \
  -kernel Image -initrd rootfs.cpio.gz \
  -append "console=ttyAMA0 rdinit=/init" -nographic
```

kvmtool 실습에 쓴 바로 그 `Image`+`rootfs.cpio.gz`를 그대로 재사용했다 — VMM만
kvmtool(KVM)에서 QEMU(TCG)로 바뀐 것. 콘솔 방식만 (`hvc0`→`ttyAMA0`) 맞춰줬다.

첫 시도는 `rdinit=/sbin/poweroff`로 깔끔한 자동 종료를 노렸는데, busybox `poweroff`가
자기 프로세스를 종료시키는 방식이라 "PID1이 죽으면 안 된다"는 커널 규칙에 걸려 패닉났다
("Attempted to kill init!"). 부팅 자체는 이미 성공한 뒤였던 예상된 부수 효과라, 원래
커스텀 `/init`으로 다시 돌려서 정확한 비교 지점을 잡았다.

비교 지점은 커널이 초기화를 다 마치고 `init`을 실행하기 직전(`Freeing unused kernel
memory` → `Run /init`) — 이 타임스탬프는 실제 경과 시간을 반영하는 가상 카운터 기반이라
TCG든 KVM이든 "게스트 안에서 실제로 몇 초가 흘렀는가"를 정직하게 보여준다.

| | 커널 초기화 완료 시점 |
|---|---|
| **KVM**(RPi5 보드, kvmtool) | `[0.087627]` — 0.088초 |
| **TCG**(WSL2, QEMU `-accel tcg`) | `[3.994910]` — 3.99초 |

**약 45배 차이.** 완전히 같은 커널 소스, 같은 부팅 작업량인데 KVM은 하드웨어에서 그대로
실행해 0.09초, TCG는 명령어 단위 번역 오버헤드로 4초 가까이 걸렸다 — "하드웨어 가속
가상화"가 실제로 만들어내는 차이의 실물 크기를 숫자로 확보했다.

왜 이렇게 차이 나는지 원리를 짚으면: TCG는 ARM의 R0/R1 같은 레지스터를 호스트 메모리의
구조체(`CPUARMState`) 필드로 취급한다 — 명령어 하나마다 이 구조체를 읽고 쓰는 번역
코드를 거친다. KVM은 게스트가 진짜 ARM 하드웨어 위에서 돌아서 R0이 **진짜 물리
레지스터**다 — 번역도 구조체 경유도 없다. 그리고 이 "레지스터=메모리 구조체"라는
특성 덕분에 TCG는 호스트가 어떤 아키텍처든 ARM을 흉내낼 수 있는 것이다(구조체는 어떤
CPU 위에서도 그냥 메모리니까) — 크로스 아키텍처가 가능한 대가로 매 명령어마다 비용을
치르는 셈이다.

## `perf kvm stat`로 VM exit 뜯어보기

게스트가 하이퍼바이저 개입이 필요한 동작(MMIO 접근, 인터럽트, WFI로 idle 진입 등)을
하려는 순간, 하드웨어가 자동으로 게스트 모드를 빠져나온다 — 이게 **VM exit**다. exit
하나하나가 컨텍스트 저장/복원 비용이 드는 진짜 오버헤드라서, `perf kvm stat`로 이걸
종류별로 세보면 "게스트가 얼마나 자주, 뭐 때문에 하이퍼바이저로 넘어오는지"가 보인다.

보드에서 게스트를 돌린 채로 `sudo perf kvm stat live -p <lkvm PID>`. 처음엔 게스트가
idle이라 `WFx`(WFI 트랩)만 잡혔다 — 게스트에서 `while true; do ls /; echo hi; done`으로
활동을 만들자 두 번째 exit 타입이 나타났다:

| VM-EXIT | Samples | Samples% | Time% | Avg time |
|---|---|---|---|---|
| WFx | 8 | 66.67% | 57.85% | 23.56us |
| DABT_LOW | 4 | 33.33% | 42.15% | 34.33us |

`WFx`는 게스트가 할 일 없어 대기하려는 순간의 트랩. `DABT_LOW`(Data Abort from Lower
EL)는 게스트가 virtio-console의 MMIO 레지스터(Stage-2에 실제 메모리로 매핑 안 된 가짜
주소)에 접근하려는 순간의 트랩 — `ls`/`echo`가 콘솔에 출력할 때마다 발생한다. 샘플
수는 WFx의 절반인데 Time% 비중은 비슷한(42% vs 58%) 걸 보면, 장치 에뮬레이션 처리가
단순 idle 대기보다 무거운 트랩이라는 것도 확인된다.

이 두 가지가 이론에서 배운 "게스트가 민감한 동작을 하면 트랩되고 유저스페이스가
처리한다"는 개념의, ARM에서의 실제 이름이자 실측값이다.

## Showcase C 완료

kvmtool 최소 게스트 부팅, vCPU 격리 코어 고정, TCG vs KVM 45배 차이, `perf kvm stat`
VM exit 분해 — 네 가지 목표 전부 달성했다. 다음 편은 이 인프라 위에서 진짜 리스크
관문을 통과한다: **취득 경로(SPI)를 격리 도메인 어디에 배치할지** 결정하는 P3다.
SMMU/IOMMU 실측부터 시작해서, 첫 시도가 P1 최악값보다 4~10배 나빴다가 원인 하나를
고치고 나서 역전되는 이야기.
