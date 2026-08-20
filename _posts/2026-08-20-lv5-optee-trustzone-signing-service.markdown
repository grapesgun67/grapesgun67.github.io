---
layout: post
title: "RPi5 하이퍼바이저 격리 드라이버 바닥부터 ⑦ — 실물 보드에서 TrustZone 서명 서비스 자작"
date: 2026-08-20 09:00:00 +0900
categories: driver linux-kernel optee trustzone arm ftrace security
series: 6
description: "STM32MP157F-DK2 실물에서 OP-TEE Trusted App을 처음부터 자작했다 — ECDSA 키는 태어난 곳(Secure World) 밖으로 한 번도 안 나간다. 그리고 그 서명 하나가 SMC를 몇 번이나 오가는지 ftrace로 직접 세봤다."
---

Lv5의 마지막 축은 격리가 아니라 **보안**이다. RPi5는 시큐어월드 지원이 약해서, 이 파트만
[Lv4에서 remoteproc 실습에 썼던]({% post_url 2026-08-09-lv4mp1-yocto-packaging-comparison %})
STM32MP157F-DK2로 무대를 옮긴다 — OP-TEE·TF-A 공식 지원 실리콘이라 QEMU가 아니라 실물에서
해볼 수 있는 기회다. 목표는 텔레메트리 서명 키를 secure storage에 보관하고 서명해주는
Trusted App(TA)을 자작하는 것.

## 출발점 — OP-TEE는 이미 살아있었다

이 보드는 이미 `TF-A BL2 → OP-TEE → U-Boot → 커널` 순서로 부팅하고 있었다 — Lv4 P3에서
`optee` flashlayout 변형으로 이미 플래싱해뒀기 때문이다. 그래서 P6은 OP-TEE OS를 새로
올리는 게 아니라, 그 위에서 TA를 자작하는 것부터 시작한다.

```bash
ls /dev/tee*                    # /dev/tee0, /dev/teepriv0 존재 확인
ps -ef | grep tee-supplicant    # tee 유저로 상시 구동 중 확인
```

둘 다 정상 — TA 개발 툴체인만 없는 상태였다.

## SDK 준비 — 정전 하나가 만든 디버깅 마라톤

TA를 자작하려면 `optee_os` 빌드 시점에만 생기는 devkit과 CA용 `libteec` 헤더가 필요하다.
이미지 재빌드+재플래싱은 무거우니, ST BSP가 제공하는 `optee-sdk` 컴포넌트를 포함한
**populate_sdk**로 독립 툴체인을 뽑는 방식을 택했다.

```bash
bitbake -c populate_sdk st-image-weston
```

그런데 빌드 도중 정전으로 강제 종료가 됐고, 재시도하니 `gdb-cross-canadian` 같은 개별
태스크들이 실패했다. `-k`(continue) 옵션으로 한 번에 전체 실패 목록을 뽑아보니, 20여
개 실패 레시피가 전부 `x86_64-nativesdk-ostl_sdk-linux`(SDK용 nativesdk) 도메인에
몰려있었다 — 태스크 종류(컴파일/설치/패키징/설정/spdx생성)는 제각각인데 도메인만
겹친다는 건, 개별 버그가 아니라 정전 당시(디스크 용량도 부족한 상태였다) 그 도메인
전체가 반쯤 쓰다 만 상태로 오염됐다는 뜻이었다. 이 도메인은 원래 이미지 빌드(Lv4 P3에서
11000개 태스크 전부 성공했던 그 빌드)엔 없던, 이번 SDK 빌드에서 처음 만들어지는
것들이라 통째로 지워도 손해가 없었다.

```bash
rm -rf tmp-glibc
find sstate-cache -type f \( -iname "*nativesdk*" -o -iname "*cross-canadian*" \) -delete
bitbake -c populate_sdk st-image-weston
```

재시도하니 이번엔 `optee-os-stm32mp` 계열에서 `QA Issue: Package version ... went
backward` 경고가 다수 떴다 — sstate 삭제 범위를 nativesdk/cross-canadian으로만 좁혔더니
타겟(arm)쪽 `optee-os` sstate는 그대로 남아있어서 새로 밀린 패키지 피드와 버전이
어긋난 것이었다. 이 QA 체크는 "실배포 패키지 피드가 다운그레이드되는 걸 막는" 용도라
1회성 로컬 빌드엔 실질적 위험이 없다 — `Tasks Summary: ... all succeeded`(진짜 태스크
실행 결과)와 `Summary: N ERROR messages`(로그 상 ERROR 카운트, setscene 실패 뒤 real
task 재실행으로 이미 복구된 것까지 포함)가 서로 다른 걸 센다는 걸 확인하고, 최종
산출물 존재로 검증했다.

```bash
find tmp-glibc/deploy/sdk -name "*.sh" -ls
# ...toolchain-5.0.17-snapshot.sh, 886MB
```

886MB 정상 크기 확인 → SDK 설치+환경 진입까지 마무리.

## 레퍼런스 예제부터 — 그리고 `libgcc.a`를 못 찾는다

자작 TA 설계 전에 공식 예제(`linaro-swg/optee_examples`)의 `secure_storage`로 TA/CA
빌드·배포 흐름부터 검증하기로 했다. 그런데 TA 링크 단계에서 막혔다:

```
arm-ostl-linux-gnueabi-ld.bfd: cannot find libgcc.a: No such file or directory
```

원인을 추적해보니, TA 빌드(`$TA_DEV_KIT_DIR/mk/link.mk`)는 일반 CA와 달리 `$CC`
(`--sysroot` 등 풀 플래그가 이미 포함된 환경변수)를 그대로 안 쓰고
`$(CROSS_COMPILE)gcc`만 맨몸으로 불렀다 — TA가 일반 userspace와 ABI가 미묘히 달라서
자체 플래그를 쓰는 게 OP-TEE 빌드시스템의 정상 설계인데, 그 과정에서 sysroot 정보가
빠지면서 컴파일러가 자기 `libgcc.a` 위치를 못 찾은 것. 실제로 `$CC
-print-libgcc-file-name`(sysroot 포함)은 정상 경로를 반환하는데, 맨몸
`arm-ostl-linux-gnueabi-gcc -print-libgcc-file-name`은 파일명만 반환하는 걸로
교차검증했다.

```bash
export CFLAGS="-B/opt/st/stm32mp1/.../usr/lib/arm-ostl-linux-gnueabi/13.4.0"
make clean && make -j$(nproc)
```

`-B<libgcc.a 디렉토리>`를 `CFLAGS`에 추가해 검색 경로를 명시적으로 주입하니 빌드가
통과했다. 보드에 배포해보니 secure storage 예제도 객체 생성→읽기→삭제까지 전부
정상 동작했다 — CA(`/dev/tee0`)→커널 optee 드라이버→SMC→OP-TEE(Secure EL1)→
TA(Secure EL0)→tee-supplicant까지 전체 경로가 실물에서 살아있다는 걸 확인한 셈이다.

## 자작 TA 설계 — 서명 서비스

여기서부터 진짜 목표: `lv5/p6-optee/signing_service/`. `secure_storage` 예제를 뼈대로
삼되, UUID만 새로 교체한 스켈레톤을 먼저 보드에서 재검증(리네임 전과 동일 동작 확인)한
뒤에야 로직을 바꾸기 시작했다 — "이름 바꾸다 깨진 것"과 "로직 바꾸다 깨진 것"을
분리하기 위한 순서였다.

설계 결정은 네 가지였다:

- **서명 대상**: TA는 페이로드의 의미를 모르는 범용 서명 서비스로 설계했다 — 텔레메트리
  구조체를 이 보드에 억지로 재현하지 않는다(MP157은 RPi5 본편과 별개 역할이라는 기존
  원칙 유지).
- **키 생성**: TA가 최초 호출 시 자체 생성해서 secure storage에 저장한다(주입 방식
  대신) — "개인키가 Secure World 밖으로 한 번도 안 나갔다"는 서사를 코드로 뒷받침하기
  위해서다.
- **알고리즘**: ECDSA P-256. 자동차 V2X 표준(IEEE 1609.2)과 동일 계열이고, 키/서명
  크기가 작아 임베디드에 적합하고, GP TEE Internal API 지원이 명확하다. HMAC(대칭키)은
  제3자 검증에 비밀 공유가 필요해서 "서명"의 핵심 가치(공개키만으로 검증)가 안 나와서
  제외했다.
- **커맨드셋**: `GENERATE_KEY`(0) / `SIGN`(1) / `GET_PUBKEY`(2) 3개. 키 생성을 SIGN에
  암묵적으로 묶지 않고 별도 커맨드로 분리해서 "키는 여기서 태어나서 안 나간다"를
  API 모양 자체로 보여주고 싶었다.

## 구현 — 코드 리뷰로 잡은 OP-TEE 특유의 함정들

TA(`ta/secure_storage_ta.c`)와 CA(`host/main.c`) 전부 직접 작성하고, 함수 하나 짤 때마다
리뷰 → 버그 지적 → 재작성 사이클로 진행했다. 잡힌 버그들이 OP-TEE 개발 특유의 함정을
잘 보여준다:

- **`generate_key()` 로직 반전**: 초안이 `TEE_OpenPersistentObject` 실패(=키 없음) 시
  에러 리턴하고, 성공(=키 있음) 시 새 키 생성을 시도하는 정반대 분기였다 — "없으면
  생성, 있으면 스킵"이 목표인데 조건이 뒤집힌 전형적 실수였다.
- **`TEE_FreeTransientObject` vs `TEE_CloseObject` 혼용**: `sign_data()`/`get_pubkey()`
  에러 처리 경로에서 **persistent 오브젝트**를 `TEE_FreeTransientObject`(transient
  전용 함수)로 닫으려던 실수 — 두 오브젝트 카테고리의 생명주기 함수가 안 섞인다는 걸
  코드로 체득했다.
- **버퍼 크기 체크 순서 오류**: `get_pubkey()`에서 CA가 준 버퍼 용량 값을 **먼저
  덮어쓰고 나서** 그 값으로 자기 자신과 비교하는 바람에 short-buffer 체크가 항상
  무력화되는 버그 — 원본 값을 먼저 비교하고 나중에 덮어써야 한다는 순서 문제였다.
- **핸들 누수**: `sign_op`/`digest_op`를 할당해놓고 이후 단계 실패 시 해제 없이
  리턴하는 경로가 여럿 있었다 — 실패 지점마다 그 시점까지 뭐가 살아있는지가 다 달라서
  매번 다시 따져야 했다.
- **CA 쪽 파라미터 누락**: `sign_data()`/`get_pubkey()` CA 함수를 처음 짤 때
  `op.params[].tmpref.buffer/size`를 아예 안 채운 채로 `TEEC_InvokeCommand`부터
  호출했다 — `TEEC_Operation`이 SMC 경계를 넘는 진짜 데이터 통로라는 걸 이때 체감했다.

## 실물 검증

<div class="demo-images">
  <figure>
    <img src="{{ '/assets/videos/lv5p6-ta-ca-output.png' | relative_url }}" alt="GENERATE_KEY→SIGN→GET_PUBKEY 실행 결과">
    <figcaption>보드에서 실행한 최종 결과 — 서명 64바이트, 공개키 64바이트</figcaption>
  </figure>
</div>

```
$ ./optee_example_secure_storage
Prepare session with the TA
Generate key (create if missing)
Sign payload: "telemetry sample data"
Signature (64 bytes): 7212b0e451f6174995e9112ee95ada90233947ba7d48c97a8dfa192da795ee...
Get public key
Public key (64 bytes): a0de507306bf23520962a31ad55cdd8da475ef92b9f62ee593a13192f4275d...
Done
```

서명 64바이트(P-256 r‖s), 공개키 64바이트(X‖Y) — 정확한 크기로 나왔다. CA→커널
드라이버→SMC→OP-TEE→TA→secure storage(암호화된 채로 Normal World 파일시스템에
저장)까지 전체 경로가 실물에서 동작한다는 걸 확인했다. 정직하게 남겨둘 한계 하나 —
서명이 API 에러 없이 리턴됐다는 것만 확인됐고, 표준 크립토 라이브러리로 독립 검증
(암호학적 정합성 증명)은 아직 다음 과제다.

## SMC 호출 경로 — kprobe 대신 커널이 이미 준비해둔 것

[Lv4에서 bpftrace 부재로 ftrace를 대체 도구로 썼던]({% post_url
2026-08-05-lv4mp1-rpmsg-driver-latency %}) 그 관례를 그대로 재사용했다. 이 커널엔
`available_filter_functions`가 없어서(`CONFIG_FUNCTION_TRACER` 미포함으로 추정)
`/proc/kallsyms`에서 후보 함수를 먼저 찾았다.

<div class="demo-images">
  <figure>
    <img src="{{ '/assets/videos/lv5p6-kallsyms-search-1.png' | relative_url }}" alt="kallsyms grep 결과 1">
    <figcaption>optee/smccc 관련 심볼 탐색</figcaption>
  </figure>
  <figure>
    <img src="{{ '/assets/videos/lv5p6-kallsyms-search-2.png' | relative_url }}" alt="kallsyms grep 결과 2">
    <figcaption>__traceiter_optee_invoke_fn_* — 정식 tracepoint를 발견한 순간</figcaption>
  </figure>
</div>

즉석에서 kprobe를 걸 필요가 없었다 — 커널에 **이미 정식 tracepoint**
(`optee_invoke_fn_begin`/`optee_invoke_fn_end`)가 있었다. `__traceiter_*`,
`trace_event_raw_event_*` 같은 TRACE_EVENT 매크로가 생성하는 심볼 세트로 확인했다.
포맷을 보니 더 좋은 소식이 있었다:

<div class="demo-images">
  <figure>
    <img src="{{ '/assets/videos/lv5p6-tracepoint-format.png' | relative_url }}" alt="tracepoint format 확인">
    <figcaption>args[8] — ARM SMCCC 레지스터 a0~a7 그 자체</figcaption>
  </figure>
</div>

`args[8]`이 곧 SMC 레지스터라, 이 tracepoint 하나로 "SMC가 정확히 이 인자값으로
발행됐다"를 직접 잡을 수 있었다.

```bash
echo 1 > /sys/kernel/debug/tracing/tracing_on
echo 1 > /sys/kernel/debug/tracing/events/optee/optee_invoke_fn_begin/enable
echo 1 > /sys/kernel/debug/tracing/events/optee/optee_invoke_fn_end/enable
echo > /sys/kernel/debug/tracing/trace
./optee_example_secure_storage
echo 0 > /sys/kernel/debug/tracing/events/optee/optee_invoke_fn_{begin,end}/enable
cat /sys/kernel/debug/tracing/trace
```

<div class="demo-images">
  <figure>
    <img src="{{ '/assets/videos/lv5p6-ftrace-capture.png' | relative_url }}" alt="ftrace 캡처 결과">
    <figcaption>실제 캡처 — begin/end 쌍이 반복되는 게 보인다</figcaption>
  </figure>
</div>

## 디코딩 — 노트북의 실제 `optee_smc.h`로 교차검증

숫자만 봐서는 아무 의미가 없어서, Yocto 빌드 트리에 있는 진짜 `optee_smc.h`를 가져와
대조했다.

- `func=3`(`0x32000003`, `0x32` 프리픽스=OP-TEE 소유 SMC 대역) = `OPTEE_SMC_FUNCID_RETURN_FROM_RPC`
  — "RPC 처리 끝났으니 재개시켜줘"
- 리턴값 `0xffff0004` = `OPTEE_SMC_RETURN_RPC_FOREIGN_INTR` — Non-secure 인터럽트(타이머
  틱 등)가 와서 잠깐 튕겨나감. **TA 로직과 무관한 노이즈**다.
- 리턴값 `0xffff0005` = `OPTEE_SMC_RETURN_RPC_CMD` — **진짜 RPC 요청**. tee-supplicant한테
  secure storage 파일 I/O 같은 실제 작업을 시키는 지점이다.
- `func=0x13`(사이클당 최초 1회) = `OPTEE_SMC_FUNCID_CALL_WITH_ARG`로 추정된다 — 문서상
  "물리주소를 넘기며 시작하는 최초 진입점" 패턴과 일치하고, 이후 전부
  `RETURN_FROM_RPC`로 그 진입을 재개하는 구조와도 맞아떨어진다(`optee_msg.h`까지
  확보하진 못해서 100% 확정은 아니다).

정리하면 실제 흐름은: **최초 진입(func 0x13) → 일하다가 인터럽트 맞으면
`RETURN_RPC_FOREIGN_INTR`로 튕겨나옴 → 처리 후 `RETURN_FROM_RPC`(func 3)로 재개 →
이걸 수백 번 반복 → 중간중간 진짜 secure storage 접근 필요하면 `RETURN_RPC_CMD`로
튕겨나와서 tee-supplicant가 실제 파일 I/O 처리 → 다시 재개 → 최종 완료.**

캡처 중에 무관한 동시 활동도 하나 섞여 있었다 — `kworker/1:1` 스레드가 별도 `param`
값으로 독립적으로 `optee_invoke_fn`을 호출하고 있었는데, kallsyms에서 봤던
`optee_rng_read`/`smccc_trng_read`(OP-TEE 기반 하드웨어 RNG 드라이버)가 백그라운드에서
엔트로피를 폴링하는 것으로 추정하고 필터링해서 제외했다.

## 정량화 — 파일 하나 여는 게 결코 SMC 한 번이 아니다

```bash
grep "optee_example_s" /sys/kernel/debug/tracing/trace > /tmp/smc_trace_ours.txt
grep -c "ffff0004" /tmp/smc_trace_ours.txt   # 145 — 인터럽트 튕겨나감(노이즈)
grep -c "ffff0005" /tmp/smc_trace_ours.txt   # 110 — 진짜 RPC
```

`GENERATE_KEY→SIGN→GET_PUBKEY` 한 세션에서 총 **255회**의 SMC 왕복이 발생했고, 그중
110회가 실제 supplicant RPC였다. `TEE_OpenPersistentObject` 한 번이 내부적으로 여러
서브 RPC(존재확인/열기/메타데이터/내용읽기 등)로 쪼개진다는 게 실측으로 드러난 셈이다.

SIGN 하나만 얼마나 무거운지 궁금해서, `main()`에서 GENERATE_KEY/GET_PUBKEY 호출을 잠깐
빼고 SIGN만 남겨 재빌드해서 다시 측정했다(이미 이전 실행으로 키가 secure storage에
있어서 GENERATE_KEY 없이도 정상 동작한다).

```bash
grep "optee_example_s" /sys/kernel/debug/tracing/trace > /tmp/smc_trace_sign_only.txt
wc -l /tmp/smc_trace_sign_only.txt        # 342줄 = 171쌍
grep -c "ffff0004" /tmp/smc_trace_sign_only.txt   # 100
grep -c "ffff0005" /tmp/smc_trace_sign_only.txt   # 64
```

**SIGN 단독으로 171회 왕복(인터럽트 100 + 진짜 RPC 64)** — 전체 세션 RPC 110회 중
**64회를 SIGN 혼자 차지**했다. GENERATE_KEY(키 이미 존재, 열기만)나 GET_PUBKEY(속성
2개 읽기)보다 SIGN이 훨씬 무겁다는 게 정량적으로 확인된 셈이다 — SHA-256 해시+ECDSA
서명 연산이 요구하는 secure storage 접근(그리고 어쩌면 ECDSA nonce용 엔트로피 요청까지)
이 실제 RPC 트래픽의 대부분을 차지하는 것으로 보인다.

## 챕터를 잠정적으로 닫으며

EL0~3 이론부터 시작해서, 정직하게 실패한 RT 베이스라인, KVM 인프라, 리스크 관문
통과, zero-copy 파이프라인, 대표 격리 실험, 그리고 이번 TrustZone 서명 서비스까지 —
"인포테인먼트가 죽어도 안전 기능은 산다"는 한 문장이 실물 RPi5와 STM32MP157F-DK2
두 보드에 걸쳐 격리와 보안 양쪽에서 증명됐다. 남은 건 SCMI 실측 노트 하나 — 이
보드가 클럭/전원 중재에 실제로 SCMI 스택을 쓰는지 확인하는, Lv5의 마지막 곁다리다.
