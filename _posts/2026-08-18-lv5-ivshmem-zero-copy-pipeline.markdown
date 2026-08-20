---
layout: post
title: "RPi5 하이퍼바이저 격리 드라이버 바닥부터 ⑤ — VM 경계를 넘는 zero-copy를 손으로 짓다"
date: 2026-08-18 09:00:00 +0900
categories: driver linux-kernel ivshmem qemu kvm pci zero-copy
series: 6
description: "Nucleo→SPI DMA→Host→ivshmem→게스트까지, 커널 모듈 없이 raw mmap으로 먼저 뚫어보고 나서 진짜 게스트 PCI 드라이버를 자작했다. UIO 대신 커스텀 캐릭터 디바이스를 고른 이유, 그리고 clock 오프셋을 지연으로 오인할 뻔한 이야기까지."
---

[4편]({% post_url 2026-08-12-lv5-isolation-domain-decision %})에서 (B) Host 취득+ivshmem
을 채택했으니, P4는 그걸 실제로 만드는 단계다. 목표는 **Nucleo(SPI DMA) → 기존 Lv2
드라이버 → Host 릴레이 → ivshmem → 게스트 커널 드라이버(자작) → 게스트 유저스페이스**를
zero-copy로 잇는 것.

## VMM 전환 — kvmtool은 여기서 손을 뗀다

kvmtool은 ivshmem을 지원하지 않는다 — 최소 구현이 목적이라 virtio-console/blk/net
정도만 있다. ivshmem은 QEMU 전용 장치라, P4부터 VMM을 QEMU(`-accel kvm`, 보드에서 실제
하드웨어 가속)로 전환했다. KVM 엔진 자체(vCPU=호스트 스레드, taskset/chrt 적용법)는
VMM이 바뀌어도 동일하다는 걸 먼저 확인했다 — [2·3편]({% post_url
2026-08-12-lv5-kvm-showcase-tcg-vs-kvm %})에서 쓴 것과 같은 `Image`/`rootfs.cpio.gz`로
QEMU+KVM 부팅이 그대로 됐다.

```bash
sudo qemu-system-aarch64 -M virt -cpu host -accel kvm -m 256 -smp 1 \
  -kernel Image -initrd rootfs.cpio.gz \
  -append "console=ttyAMA0 rdinit=/init" \
  -object memory-backend-file,id=ivshmem-mem,size=1M,mem-path=/dev/shm/ivshmem_telemetry,share=on \
  -device ivshmem-plain,memdev=ivshmem-mem \
  -nographic
```

게스트에서 vendor/device ID(`0x1af4 0x1110`)로 ivshmem-plain 장치를 특정했다.
`resource2`가 정확히 1048576바이트(1MB, 요청한 그대로)로, 이게 진짜 공유메모리 BAR다.

## 드라이버 짜기 전에 raw mmap으로 먼저 뚫어본다

`dd`/`cat`으로 `resource2`를 읽으려 하니 **"Input/output error"**가 났다. 커널이 이런
큰 prefetchable 메모리 BAR은 `read()`/`write()` syscall을 아예 막아두고 `mmap()`만
허용하기 때문이다(`dd`는 내부적으로 read/write만 쓴다). 그래서 최소 진단 도구
`mmapdump.c`(mmap으로 열어 hexdump, 필요시 쓰기도 지원)를 새로 작성해 aarch64 정적
바이너리로 크로스컴파일했다 — 이건 `devmem`/`devmem2`, DPDK의 UIO 기반 접근과 같은
"드라이버 짜기 전에 raw mmap으로 하드웨어부터 확인"하는 실전 브링업 관행이다.

Host에서 `/dev/shm/ivshmem_telemetry`에 문자열을 직접 쓰고, Guest에서 `mmapdump`로
`resource2`를 읽으니 **동일한 문자열이 트랩 없이 그대로 보였다.** Stage-2가 이 영역을
진짜 물리메모리로 직접 매핑해준다는 걸 실측으로 확인한 것 — [이전 편에서 이론으로 정리한
"virtio는 트랩, ivshmem은 무트랩" 구조]({% post_url 2026-08-12-lv5-kvm-showcase-tcg-vs-kvm
%})가 실전에서 그대로 재현됐다.

## Host 릴레이 — seqlock을 VM 경계 너머로 승격

[Lv2의 `kim_spi_shared` 구조체]({% post_url 2026-07-21-lv2-mmap-seqlock %})를 그대로
재사용해 ivshmem 공유 영역의 레이아웃으로 삼았다. `host_relay.c`가 하는 일:

- `/dev/kim-spi`를 mmap(Lv2의 seqlock 읽기 재시도 루프 재사용)
- `/dev/shm/ivshmem_telemetry`를 mmap(QEMU가 이미 만들어둔 파일에 붙는다)
- 소스 `seq`가 바뀔 때만 ivshmem 쪽에 동일한 seqlock 프로토콜로 재발행
- **`read()`/`write()` syscall 전혀 없이 mmap 이후 순수 메모리 접근만으로 릴레이**

실행해보니 `published`/`last_seq` 카운터가 `retries=0`으로 꾸준히 증가했다(찢어진 읽기
없이 seqlock 정상 동작). Guest에서 `mmapdump`로 반복 관찰하니 seq/데이터 값이 시간에
따라 실제로 바뀌었다 — **커널 드라이버 하나 없이 전체 경로가 이미 살아있다**는 걸
증명한 것. 여기까지가 "raw 메커니즘 검증"이고, 이제 이 위에 진짜 게스트 커널 드라이버를
얹을 차례다.

## 게스트 드라이버 — UIO 대신 커스텀 캐릭터 디바이스를 고른 이유

가장 먼저 UIO(Userspace I/O, `drivers/uio/uio_pci_generic.c`)와 커스텀 캐릭터 디바이스
중에 저울질했다. 웹 리서치 결과 UIO는 **IOMMU 보호가 없고 DMA 가능 장치엔 애초에
안전하지 않게 설계돼있으며**(root 전용, 시큐어부트 환경에선 커널이 아예 막기도 함),
실무에서도 "간단한 경우"에 한정된 지름길이고 더 진지한 용도엔 VFIO나 커스텀 드라이버가
표준이라는 근거를 확인했다 — [Lv2의 `kim-spi.c`]({% post_url
2026-07-19-lv2-char-driver-irq %})와 동일한 커스텀 캐릭터 디바이스 방식을 채택했다.
일관성도 챙기고 실전 드라이버 작성 경험도 하나 더 쌓는 선택이었다.

구조는 대부분 Lv2 패턴을 재사용했다:
- `pci_device_id` 테이블에 실측한 vendor/device ID(`0x1af4:0x1110`) 등록 → 자동 매칭
- `pcim_enable_device`+`pcim_iomap_region`으로 BAR2 매핑(커널 소스에서
  `pcim_iomap_regions`(복수형)이 **DEPRECATED**로 마킹된 걸 확인하고 단수형 함수 사용)
- 캐릭터 디바이스 등록은 `kim-spi.c`와 동일한 패턴+goto 기반 에러 역순 정리
- `.open`: `container_of`로 디바이스 구조체 복원(`kim-spi.c`와 완전히 동일)
- `.mmap`: **`remap_pfn_range`**로 BAR2 물리주소를 유저 VMA에 매핑 — Lv2는
  `vm_insert_page`를 썼는데, 그건 `alloc_page`로 만든 진짜 RAM 페이지용이고 여기는 PCI
  BAR라는 MMIO 물리주소라서 다른 API가 필요했다.

코드 리뷰에서 실제로 잡힌 이슈 둘: 변수명 오타(`idev`↔`priv` 혼용, 컴파일 에러로 바로
드러남)와, **`vma->vm_pgoff`를 무시하는 버그** — 유저가 mmap에 0이 아닌 offset을 주면
조용히 엉뚱한 주소를 매핑해주는 잠재적 문제였다. 이 장치는 offset 0만 지원하니
`if (vma->vm_pgoff != 0) return -EINVAL;`로 명시적으로 거부하도록 고쳤다.

## 실기 검증 — 그리고 진단 도구가 스스로 사고를 낸 해프닝

`insmod kim_ivshmem.ko` → dmesg에 `probe` 로그(vendor/device ID, BAR2 phys/len 정확히
일치) → `/dev/kim_ivshmem` 캐릭터 디바이스 노드 생성 → `mmapdump /dev/kim_ivshmem 16`으로
실제 읽기. **seq가 계속 증가하고 데이터(float)도 실시간으로 바뀌는 걸 확인** — Host
릴레이가 계속 쓰고 있는 실제 Nucleo 텔레메트리가 자작 드라이버를 통해 그대로 보였다.

디버깅 중 재미있는 해프닝이 하나 있었다. 터미널에 명령어가 겹쳐 쳐져서
(`mmapdump ... 16mmapdump ... 16`) `mmapdump`의 인자 파싱이 꼬여 의도치 않게 "쓰기
모드"가 발동했고, 진단 도구 자신이 공유메모리에 문자열을 써버려 데이터가 깨지는 일이
있었다 — **seqlock 없이 막 쓰면 이렇게 된다는 걸 실수로 직접 실증한 셈**이다. 인자
개수를 역산해서 원인을 진단하고, 명령어를 깔끔하게 재입력해서 해결했다.

## 정량화 — 복사 3번, Guest 쪽은 0번

**복사 횟수**: SPI 컨트롤러의 DMA 쓰기(하드웨어가 함, 집계 제외)를 빼면 CPU `memcpy`는
정확히 3번이다 — (1) `kim-spi.c`: DMA 스테이징 버퍼 → seqlock 공유 페이지, (2)
`host_relay.c`: 공유 페이지 → 로컬 버퍼, (3) `host_relay.c`: 로컬 버퍼 → ivshmem 공유
영역. **ivshmem에 들어간 뒤로는 복사가 0번** — `.mmap`의 `remap_pfn_range`가 BAR2
물리주소를 게스트 유저스페이스 주소공간에 직접 매핑해줘서, VM 경계를 넘는 마지막
구간은 포인터 연결일 뿐 데이터 복제가 없다. 이게 "zero-copy"의 실체다.

**지연 측정**은 두 가지 지표를 따로 뒀는데, 신뢰도가 다르다:

- **inter-arrival**(게스트 자기 클럭 하나로만 잼, 신뢰 가능): 새 seq를 감지한 시각들
  사이의 간격. 50회 측정 결과 평균 3477us — Lv2에서 실측한 Nucleo 배치 주기(~3.4ms)와
  거의 정확히 일치했다. 이건 병목이 폴링 속도가 아니라 **실제 소스(Nucleo) 갱신 주기**
  라는 뜻이다.
- **latency**(Host/Guest 서로 다른 클럭 비교, 절대값 신뢰 불가): 평균 325ms. 값 자체는
  비정상적으로 컸지만, 50개 샘플이 4~5ms 폭의 아주 좁은 범위에 몰려있었다 — 무작위
  지터가 아니라 **상수 오프셋**의 특징이다. 처음엔 `date +%s`로 초 단위 비교했을 때
  값이 같아서 "동기화됐다"고 오판했는데, 이건 서브초 오프셋을 구분할 정밀도가 없어서였다
  (같은 정수 초 안에 다 들어가버림). 진짜 원인은 kvmclock이 게스트 시계의 "속도"(드리프트)
  는 지속적으로 보정하지만, 부팅 시점의 "절대 시각"은 별개 메커니즘이라 최소 커널+최소
  rootfs 게스트에서 이 절대 동기화가 부실했을 가능성으로 진단했다. 그래도 절댓값은
  못 믿어도 **편차(최댓값-최솟값 ≈ 4~5ms)는 상수 오프셋이 상쇄되고 남는 실제 지연
  변동폭**으로 해석할 수 있었고, 이론적 기대치(게스트 폴링 0.2ms + 소스 주기 이내)와
  정합적이었다.

디버깅 중 `host_relay`가 `ps aux`에서 3개 PID로 보여서 중복 실행을 의심한 적도 있는데,
이 시스템의 `sudo`가 `use_pty` 옵션이라 단일 실행도 여러 pts로 분산 표시되는 정상
현상이었다 — 예전 kvmtool 조사 때 봤던 "sudo→sudo-monitor→lkvm" 3단 구조와 같은 패턴.

## 임의 설계가 아니었다 — 업계 사례로 확인

`kim-spi`(드라이버)/`host_relay`(중계)/`ivshmem`(공유메모리)/게스트 드라이버(소비)로
나눈 4단 구조가 임의의 선택이 아니라는 것도 리서치로 확인했다:

- **AUTOSAR PduR(PDU Router)**: 자동차 CAN 스택 표준 자체가 "드라이버"와 "라우팅/
  게이트웨이(데이터 내용은 안 건드리고 어디로 보낼지만 결정)"를 분리하도록 명세돼있다 —
  `kim-spi`/`host_relay`(seqlock으로 그대로 복사만 하고 데이터 해석 안 함) 구조와 같은
  분리 원칙이다.
- **OpenSynergy COQOS Hypervisor SDK**(르네사스 R-Car H3, 실제 양산 SoC): IVI/클러스터
  VM과는 별도로 AUTOSAR 호환 CAN 게이트웨이가 별도 코어에서 도는 구조 — `host_relay`를
  별도 프로세스로 분리한 것과 같은 발상이다.
- **EB tresos Embedded Hypervisor**: "프록시 모듈"이라는 공식 용어로 소스와 공유메모리
  사이에 중계 컴포넌트를 두는 패턴을 문서화하고 있다 — `host_relay`가 정확히 이 역할이다.

## 다음

파이프라인은 완성됐다. 다음 편은 이 위에서 이 프로젝트의 대표 실험을 돌린다 — Host에
`stress-ng`를 걸어서 인포테인먼트가 무너져도 격리 도메인의 텔레메트리 경로는 살아있는지,
지터로 직접 증명한다.
