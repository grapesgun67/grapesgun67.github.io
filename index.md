---
layout: home
title: "QEMU로 PCIe 가속기 드라이버 바닥부터"
---

QEMU 위에서 가상 PCIe 장치를 설계하고(pre-silicon 방식), 그 Linux 커널 드라이버와
유저스페이스 런타임을 바닥부터 구현·디버깅한 기록. edu 장치 해부부터 시작해서
내 장치(vecadd)를 직접 저작하고, char device / ioctl / mmap으로 유저스페이스까지
연결하는 5부작.

소스: [github.com/grapesgun67/bsp_device_driver](https://github.com/grapesgun67/bsp_device_driver)
