---
layout: post
title: "카메라/ISP V4L2 드라이버 바닥부터 ① — 이론과 실측이 갈라지는 지점들"
date: 2026-07-29 12:00:00 +0900
categories: driver linux-kernel v4l2 media-controller camera
series: 3
description: "I2C는 되는데 CSI-2만 타임아웃, 포맷은 맞춘 것 같은데 EINVAL, 그다음엔 에러도 없이 영원히 멈추는 스트리밍 — Raspberry Pi Camera Module 3(IMX708)에서 첫 프레임을 뽑기까지 세 번 막힌 이야기."
---

Lv1(QEMU 가상 PCIe 장치)과 Lv2(RPi5↔Nucleo SPI)는 디바이스 노드 하나로 끝났다. 카메라는
다르다 — 센서·CSI 리시버·ISP가 각자 별개의 실리콘 블록이고, 이걸 표준 서브시스템(V4L2)이
**그래프**로 다뤄야 해서 인터페이스 자체가 세 층으로 쪼개진다.

## 카메라가 왜 노드 하나가 아닌가

**Media Controller**(`/dev/media0`)가 센서·CSI-2 리시버·ISP를 엔티티+링크로 표현하고,
유저스페이스가 그 그래프를 열거·재구성한다. subdev(`/dev/v4l-subdevN`)는 포맷·컨트롤
설정 전용(control-plane), 실제 픽셀은 video device(`/dev/videoN`)로만 흐른다
(data-plane). 순서도 강제된다 — **링크(어느 경로가 활성인지) → 포맷(그 경로를 따라
전파) → 스트리밍** 순서가 아니면 안 된다. 링크가 안 정해진 상태에선 포맷을 뭘 따라
전파할지 자체가 불명확하기 때문이다.

이 구조는 소프트웨어가 임의로 나눈 게 아니라, 하드웨어 배선 단계부터 이미 그렇게
갈라져 있다 — 센서를 설정하는 I2C 채널(저속 2선)과 픽셀이 흐르는 CSI-2 채널(고속
차동신호)은 물리적으로 완전히 다른 배선이다. 이 사실이 첫 번째 디버깅에서 그대로
드러났다.

## 디버깅 ① — I2C는 되는데 CSI-2만 죽어있다

`dmesg`상으로는 다 정상이었다. `imx708 10-001a: camera module ID 0x0301`(I2C 레지스터
읽기 성공), `rpicam-hello --list-cameras`도 센서를 정상 인식. 근데 실제 캡처
(`rpicam-still`)를 시도하면 `/dev/video4`에서 1초간 프레임이 하나도 안 들어와
`Dequeue timer ... has expired!` → `Camera frontend has timed out!`로 실패했다.

I2C(제어 평면)는 살아있는데 CSI-2(데이터 평면)만 죽어있는 비대칭 실패 — 위에서 얘기한
"제어/데이터 평면이 물리적으로 분리된 배선"이라는 그림이 실측으로 그대로 갈라져
나타난 형태다. 고속 차동신호 레인은 접촉 마진에 훨씬 예민하니 케이블이 유력 후보였다.
FPC 케이블 양쪽 래치를 완전히 열고 끝까지 평행하게 다시 꽂으니 즉시 30fps로 정상
스트리밍됐다.

**교훈**: CSI 카메라 브링업에서 "I2C는 되는데 캡처만 실패"는 배선 문제로 삽질 범위를
좁혀주는 신호다.

## 디버깅 ② — 포맷을 맞췄는데 `EINVAL`

Phase A의 다른 목표는 `libcamera` 없이 `media-ctl`/`v4l2-ctl`만으로 raw 캡처를
해보는 것이었다. raw bypass 경로(`csi2:4 → rp1-cfe-csi2_ch0`, `/dev/video0`)에
센서 pad0 기준 포맷(`pBAA`, 10bit packed)을 설정하고 `STREAMON`했더니 즉시
`-22`(`EINVAL`), `dmesg`엔 `Format mismatch! Code 12317 fourcc pBAA`.

`0x301d`(=12317)를 커널 헤더(`media-bus-format.h`)에서 찾아보니
`MEDIA_BUS_FMT_SBGGR16_1X16`이었다 — **`/dev/video0`가 실제로 연결된 건 csi2의 pad4고,
그 pad의 포맷은 센서 pad0(10bit)가 아니라 16bit 컨테이너**였다. 링크 저편(csi2 pad4)의
실제 포맷 대신 센서 자신의 포맷만 보고 판단한 게 원인 — `pixelformat=BYR2`(16bit
Bayer)로 바꾸니 해결됐다.

## 디버깅 ③ — 에러도 없이 영원히 멈추다

포맷을 맞추고 다시 `STREAMON`했더니 이번엔 에러도 안 나고 그냥 프레임이 영원히 안
들어왔다. `lsmod`로 확인해보니 실제 바인딩된 드라이버가 문서에서 흔히 보는
`rp1-cfe`가 아니라 **`rp1_cfe_downstream`**이었다(둘 다 커널 설정엔 `=m`으로 켜져
있었는데 이쪽이 실제로 물림). 소스(`drivers/media/platform/raspberrypi/rp1_cfe/cfe.c`)
의 `cfe_start_streaming()`을 직접 읽었다:

```c
if (!test_all_nodes(cfe, NODE_ENABLED, NODE_STREAMING)) {
    cfe_dbg("Not all nodes are set to streaming yet!\n");
    return 0;   // 센서에 s_stream(1) 호출 자체를 안 함
}
```

`test_all_nodes(precond, cond)`의 실제 의미는 **"링크가 ENABLED인 노드는 전부 다 지금
STREAMING 중이어야 한다"**는 전칭 조건이었다. 이전에 `libcamera`를 한 번 돌렸던 흔적으로
`rp1-cfe-embedded`·`rp1-cfe-fe_stats`·`rp1-cfe-fe_config`·`rp1-cfe-fe_image0` 링크가
ENABLED로 남아있었는데, 이번엔 ch0 하나만 스트리밍하려니 이 조건을 통과하지 못해
센서가 영원히 안 깨어난 것이었다. `media-ctl -l`로 이 4개 링크를 전부 disable하고
ch0만 남기니 즉시 해결 — `raw_bypass.raw` 23,887,872바이트(=4608×2592×2, 계산과 정확히
일치) 캡처 성공.

이건 공식 문서(`rp1-cfe.rst`)에도 안 나오는, `rp1_cfe_downstream` 변종 고유의 동시성
계약이었다 — 소스를 직접 안 읽었으면 못 찾았을 버그다.

## 결과

`libcamera`/`rpicam-still`로 정상 디베이어된 사진 한 장, 그리고 `libcamera` 없이
`media-ctl`/`v4l2-ctl`만으로 raw 캡처 한 장 — 두 경로 다 성공했다. 이론(3층 모델)과
실측 토폴로지도 거의 일치했는데, `imx708`이 pad를 2개(진짜 이미지 + embedded data) 갖고
있다거나, `csi2`의 source pad 하나가 두 다운스트림(raw bypass / 처리 경로)으로 동시에
fan-out 가능하다는 건 실측에서 새로 드러난 디테일이었다.

다음 편은 이 캡처 버퍼가 완전히 다른 커널 서브시스템(PiSP Back End)과 복사 없이
공유되는 과정 — 그리고 카메라를 안 쓸 때 전원을 어떻게 끄는지에 관한 이야기다.
