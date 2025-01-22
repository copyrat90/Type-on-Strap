---
layout: post
title: GBA를 USB 컨트롤러로 만드는 프로젝트 (gba-pico-gamepad)
tags: [GBA]
author: copyrat90
last_modified_at: 2023-05-28T12:00:00+09:00
---

[프로젝트 GitHub 링크](https://github.com/copyrat90/gba-pico-gamepad)

[시연 동영상 (유튜브)](https://www.youtube.com/watch?v=JmBufgcb4Gw)

[![](https://img.youtube.com/vi/JmBufgcb4Gw/hqdefault.jpg)](https://www.youtube.com/watch?v=JmBufgcb4Gw "시연 동영상 (유튜브)")

# 프로젝트 설명

이 프로젝트는 [Game Boy Advance (GBA)](https://namu.wiki/w/GBA)를 USB 게임 컨트롤러로 만드는 프로젝트다.\
라즈베리파이 피코를 인터페이스 장치로 활용하였다.

GBA에는 다른 GBA와 통신하여 멀티 플레이 게임을 할 수 있도록 시리얼 포트가 있고,\
해당 규격에 맞는 전용 통신 케이블이 있다.\
또한 GBA는 SPI 통신을 지원하므로, GBA 외에 다른 장치와의 통신도 쉽게 할 수 있다.

![GBA pinout](https://user-images.githubusercontent.com/1631752/124884342-8ee7fc80-dfa8-11eb-9bd2-4741a4b9acc6.png "GBA pinout")

# 준비물

1. GBA와 GBA 통신 케이블
    * 통신 케이블에 GND 핀이 없는 경우가 있다.
      다인용 포트가 있는 케이블이 GND 핀이 있을 확률이 높다.
2. 라즈베리파이 피코와 5핀 USB 케이블
3. 점퍼 케이블 (M to F) 4개, 절연 테이프와 가위

SPI 통신을 위해 GBA 통신 케이블을 잘라서 그림과 같이 라즈베리파이 피코에 연결한다.\
GBA 통신 케이블 색상은 제조사에 따라 랜덤일 것이므로, 케이블 헤드를 잘라서 내부를 확인해보는 게 좋을 것이다.

[자세한 사용 방법](https://github.com/copyrat90/gba-pico-gamepad#usage)

![Overall pinout](https://i.ibb.co/xgZW66y/rpi-pico-pinout.png "Overall pinout")

# 작성한 프로그램

GBA의 키 입력을 USB 컨트롤러 신호로 변경하기 위해서는, 2개의 프로그램이 필요하다.

1. 통신 케이블로 GBA 키 입력을 보내는 프로그램 (GBA에서 동작)
2. 그 신호를 받아서 USB 게임 컨트롤러 입력 신호로 변경하여 전송하는 프로그램 (라즈베리파이 피코에서 동작)

## 2. Pico → USB 키입력 전송

2번째 프로그램을 작성하기 위해, 라즈베리파이 피코 전용 오픈 소스 게임패드 펌웨어인\
[GP2040-CE](https://github.com/OpenStickCommunity/GP2040-CE) 를 활용하였다.

원래 GP2040-CE는 각 GPIO 핀에 버튼 1개씩을 할당하여 Pull-up 신호를 받아 USB 컨트롤러 신호로 변경하여 전송하는 기능을 한다.\
하지만, GBA에는 총 10개의 버튼이 있고, 통신 케이블에는 GND와 Vcc 핀을 제외하면 통신용 핀이 4개뿐이다.\
그러므로 GPIO 핀에 버튼 1개씩을 할당하는 기존 펌웨어를 그대로 사용할 수는 없다.

그래서 [GP2040-CE의 소스 코드를 수정](https://github.com/copyrat90/gba-pico-gamepad/compare/aed5cb1..be230df#diff-3daa7b69467513f29fa16f889809525f7328a67ecd079238a5bbfd1890f0e3c2)하여, 각 GPIO 핀 당 하나씩 버튼 입력을 받는 대신\
SPI 통신을 이용해 버튼 입력을 받을 수 있도록 하였다.

## 1. GBA → Pico 키입력 전송

다음으로, GBA에서 동작하는 1번째 프로그램을 만들기 위해서,\
GBA의 시리얼 포트를 추상화하는 [gba-link-connection](https://github.com/rodri042/gba-link-connection) 라이브러리를 사용하였다.

마침 gba-link-connection에 포함된 SPI 통신 예제가 키 입력을 보내는 예제여서, [이를 조금 수정해 사용](https://github.com/copyrat90/gba-pico-gamepad/blob/be230dfe57a7a0e944f11893fb6fd2b004cfc627/gba-link-connection/examples/LinkSPI_demo/src/main.cpp)하였다.

해당 프로그램은 GBA의 장치 입출력(I/O) 레지스터 중\
키패드에 해당하는 I/O 레지스터의 값을 읽어 SPI 통신으로 전송한다.

![`REG_KEYS` I/O 레지스터](/assets/img/posts/2023-05-28-gba-pico-gamepad/gba-key-input-registers.png "`REG_KEYS` I/O 레지스터")

해당 I/O 레지스터는 위와 같이 구성되므로,\
2번째 프로그램에서 전송받은 키를 확인할 때 확인할 키의 값을 비트 연산을 통해 가져온다.

## 0. Pico → GBA 프로그램 전송

마지막으로, 작성한 GBA 프로그램을 실제로 GBA에 넣어 동작시킬 방법이 필요하다.\
일반적으로는 별도의 게임 팩(카트리지)이 필요하지만, 다른 더 좋은 방법이 있다.

GBA에는 게임 팩 1개로 여러 GBA에서 멀티 플레이 게임을 즐길 수 있도록\
시리얼 포트를 통해 프로그램을 전송하는 기능이 있다.\
GBA에서는 이를 [Multiboot 기능](http://problemkaputt.de/gbatek-bios-multi-boot-single-game-pak.htm)이라고 부른다.\
이를 이용해, 라즈베리파이 피코에 2번째 프로그램을 저장해뒀다가 GBA로 전송하여 시작할 수 있다.

Multiboot 프로토콜에 맞게 프로그램을 전송하기 위해, 라즈베리파이에서 이를 수행하는\
[gba_03_multiboot](https://github.com/akkera102/gba_03_multiboot) 오픈 소스 프로그램의 코드를 활용하였다.\
이를 라즈베리파이 피코 환경에 맞게 수정해, [함수로 만들어 펌웨어 코드에 추가](https://github.com/copyrat90/gba-pico-gamepad/blob/be230dfe57a7a0e944f11893fb6fd2b004cfc627/GP2040-CE/src/gba/multiboot.cpp)하였다.

라즈베리파이 피코에 1번째 프로그램을 저장하기 위해,\
1번째 프로그램을 먼저 빌드하고, 빌드된 바이너리를 `constexpr unsigned char` 배열 형식으로 만들어\
펌웨어에 임베딩하는 [빌드 스크립트](https://github.com/copyrat90/gba-pico-gamepad/blob/be230dfe57a7a0e944f11893fb6fd2b004cfc627/build.sh#L10-L11)를 작성했다.

이를 펌웨어의 main() 함수 시작시에 불러와 [GBA로 전송](https://github.com/copyrat90/gba-pico-gamepad/blob/be230dfe57a7a0e944f11893fb6fd2b004cfc627/GP2040-CE/src/main.cpp#LL29C3-L29C3)한다.

# 마치며

설명한 내용 외에도, 오작동을 방지하기 위해 펌웨어 코드를 추가로 수정한 부분이 더 있으나,\
중요하지 않은 관계로 생략하였다.

*마지막 수정 : {{ page.last_modified_at }}*
