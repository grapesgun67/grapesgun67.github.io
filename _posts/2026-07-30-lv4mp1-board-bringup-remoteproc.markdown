---
layout: post
title: "STM32MP157F-DK2로 넘어와서 ① — 새 보드, 새 네트워크, 그리고 remoteproc 첫 성공"
date: 2026-07-30 09:00:00 +0900
categories: driver linux-kernel arm stm32mp1 amp remoteproc openamp
series: 5
description: "Zynq-7000엔 remoteproc가 없다는 걸 확인하고 STM32MP157F-DK2로 보드를 옮겼다. 노트북을 빌드서버로 세팅하는 것부터, ST 공식 예제 하나로 A7↔M4 RPMsg 왕복을 첫 시도에 성공시키기까지."
---

[3편]({% post_url 2026-07-29-lv4-kernel-module-and-pivot %})에서 Zynq-7000엔 mainline에도
Xilinx 벤더 커널에도 remoteproc 드라이버가 없다는 걸 소스로 확인하고, STM32MP157F-DK2로
보드를 옮기기로 했다. 이 칩은 Cortex-A7(리눅스)과 Cortex-M4(베어메탈/FreeRTOS)가 한
다이 안에 있고, mainline이 `stm32_rproc`/`stm32-ipcc`를 정식 지원한다 — Zynq 세 편에서
손으로 재현했던 것들(코어 기동, 크로스코어 시그널링, 공유메모리)을 이번엔 표준 프레임워크가
대신 해준다.

## 빌드 환경부터 — 메인 PC 대신 노트북

Yocto/OpenSTLinux 빌드는 80GB 이상의 디스크와 몇 시간의 컴파일 시간이 필요한데, 메인
PC는 그럴 공간이 없었다. 노트북을 사실상 빌드서버로 쓰기로 했는데, 문제가 하나 있었다 —
메인 PC는 폰 테더링, 노트북은 와이파이로 각자 다른 사설망에 있어서 서로 통신이 안 됐다.

랜선으로 두 기기를 직결하고 고정 IP(`192.168.123.1`/`.2`)를 부여했는데, 처음엔 ping조차
안 됐다. 방화벽 커스텀 규칙을 의심했지만 진짜 원인은 훨씬 단순했다:

1. **네트워크 프로필이 "공용"으로 잡혀 있었다** — Windows 방화벽은 도메인/개인/공용
   세 벌의 규칙셋을 따로 갖고 있고, 공용은 기본적으로 들어오는 연결 대부분을 막는다.
   `Set-NetConnectionProfile -NetworkCategory Private`로 강제 전환.
2. **"파일 및 프린터 공유"가 꺼져 있었다** — 이게 단일 스위치가 아니라 SMB/NetBIOS/
   프린트스풀러/**ICMP 에코 요청(ping)**을 묶은 규칙 그룹이라는 걸 몰랐다. 개인 열만
   체크(공용 열은 낯선 와이파이 노출 방지를 위해 의도적으로 미체크).

WSL2는 기본 NAT 모드에서 자기만의 가상 IP 뒤에 숨어있어서 노트북 밖에서 안 보인다 —
`.wslconfig`에 `networkingMode=mirrored`를 켜서 Windows 호스트와 같은 IP를 그대로
공유하게 만들고, SSH 키인증까지 세팅했다. 이 구성 위에 툴체인을 완전히 분리했다: Yocto
빌드는 노트북 WSL2(순수 컴파일, USB 안 건드림), STM32CubeIDE/STM32CubeProgrammer는
노트북 네이티브 Windows(ST가 진짜 Windows 실행파일을 배포해서 USB 패스스루가 아예
불필요).

## OpenAMP_TTY_echo 예제 딥다이브 — 손으로 만들기 전에 먼저 읽는다

보드가 오기 전, ST 공식 예제 `OpenAMP_TTY_echo`(CubeMP1 번들)를 소스 레벨로 먼저
읽었다. 몇 가지가 Zybo 때 배운 것과 정확히 연결됐다:

- **링커스크립트의 `m_ipc_shm`**(`ORIGIN=0x10040000, LENGTH=0x8000`)이 디바이스트리의
  `vdev0vring0@10040000`/`vdev0buffer@10042000`과 물리주소로 정확히 맞물린다 — Zybo
  P1에서 손으로 배치했던 `0x10000000`/`0x10100000`과 개념적으로 동일한 "링커스크립트와
  DT가 미리 악수해두는" 패턴.
- **리소스 테이블**(`rsc_table.c`)은 `.resource_table`이라는 ELF 특수 섹션에 배치되는
  C struct — `strip`해도 심볼 이름만 사라지지 이 섹션 자체는 안 지워진다는 것도 확인.
- **vring**은 virtio 표준으로 못박힌 정확한 바이트 오프셋 구조라서, 서로 다른 회사가
  각자 컴파일한 A7 커널과 M4 펌웨어가 소스 공유 없이도 같은 메모리를 해석할 수 있다.

## 보드 도착, 그리고 두 번의 실전 트러블슈팅

칩 마킹을 확인해보니 `STM32MP157FAC1` — 크립토 지원 F 변형이 실제로 맞았다(주문 전
데이터시트 검색 스니펫으로만 판단했던 걸 실물로 확정). 플래싱은 두 군데서 걸렸다:

- **WSL2→Windows 복사 시 심볼릭 링크가 0바이트로 깨졌다.** Yocto 빌드 산출물엔 심볼릭
  링크가 많은데, 일반 `cp`로는 WSL/Windows 경계를 넘으면서 링크가 실제 파일 내용 없이
  빈 파일이 됐다. `cp -rL`(링크를 실제 파일로 풀어서 복사)로 해결.
- **`arm-trusted-firmware`/`fip` 서브폴더를 빠뜨렸다.** 처음엔 `.ext4` 이미지 파일들만
  옮겼는데, `deploy/images/stm32mp1/` 전체 트리를 통째로 옮겨야 했다.

BOOT0/BOOT2 DIP 스위치는 공식 표(0/0=DFU, 1/1=SD)와 실물 라벨 방향이 안 맞았다 —
"DFU로 연결했던 것과 같은 물리 위치"가 실제로는 정상 SD 부팅이었다. 데이터시트보다
실측을 우선했다. 첫 부팅은 TF-A BL2 → OP-TEE → U-Boot → 커널 6.6.129까지 전체 체인이
한 번에 성공했다.

## P3 핵심 목표 — remoteproc 로드 + RPMsg 왕복, 첫 시도에 성공

`OpenAMP_TTY_echo` 예제를 원본 그대로(수정 없이) CubeIDE로 빌드해서 `.elf`를 만들고,
`scp`로 보드에 올린 다음:

```
echo OpenAMP_TTY_echo_CM4.elf > /sys/class/remoteproc/remoteproc0/firmware
echo start > /sys/class/remoteproc/remoteproc0/state
```

커널 로그가 전체 체인을 그대로 보여줬다:

```
remoteproc remoteproc0: powering up m4
Booting fw image OpenAMP_TTY_echo_CM4.elf, size 2502780
rproc-virtio rproc-virtio.2.auto: assigned reserved memory node vdev0buffer@10042000
virtio_rpmsg_bus virtio0: creating channel rpmsg-tty addr 0x400
virtio_rpmsg_bus virtio0: creating channel rpmsg-tty addr 0x401
rproc-virtio rproc-virtio.2.auto: registered virtio0 (type 7)
remoteproc remoteproc0: remote processor m4 is now up
```

`vdev0buffer@10042000`이 실제로 배정되는 걸 보고, 미리 읽어뒀던 링커스크립트의
`m_ipc_shm`과 DT의 예약메모리가 정확히 맞물린다는 걸 실물로 확인했다. `type 7`도
`rsc_table.c`의 `VIRTIO_ID_RPMSG` 값과 일치. `/dev/ttyRPMSG0`/`1`이 생겼고, echo
테스트도 그대로 돌아왔다:

```
echo "Hello Virtual UART0" >/dev/ttyRPMSG0   # → 그대로 되돌아옴
```

A7→IPCC→M4(콜백에서 echo)→IPCC→A7, 전체 왕복 경로가 실증됐다. Zynq에서 세 편에 걸쳐
겨우 도달했던 지점(코어 간 통신)을, 이번엔 표준 프레임워크 덕분에 첫 시도에 도달했다 —
다만 이건 아직 ST 예제를 그대로 쓴 것뿐이다. 다음 편은 이 채널을 우리가 직접 쓴 커널
드라이버로 바꾸고, 왕복 지연을 실측하는 이야기다.
