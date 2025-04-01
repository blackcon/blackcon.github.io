---
title: CTF 문제로 알아보는 ESP32 리버싱
categories: [Hacking, CTF]
tags: [reversing, esp32, hardware]
date: 2025-03-31 20:00:00 +0900
---

Codgate 2025에서 출제된 리버싱 문제 `physical`을 통해 ESP32 바이너리를 분석해보는 과정을 정리한 글이에요. 이번 분석에서는 flash 이미지(`flash.bin`)로부터 애플리케이션 코드(`app0`)와 크래시 상황을 담은 `coredump`를 추출하고, Ghidra와 다양한 분석 도구를 활용해 그 내용을 살펴봤어요.

[ESP32](https://namu.wiki/w/ESP32)는 MCU 기반의 SoC(System-on-Chip)로, IoT 기기에서 자주 사용되며 내부 flash에 파티션 형태로 데이터를 저장하고 실행해요. CTF 문제로 출제되었을 경우, 일반적으로 다음과 같은 흐름으로 분석이 진행됩니다:

> flash 영역 분석 → app 코드 추출 → coredump 추출 → 리버싱 및 Emulation

---

## 📦 flash.bin 파티션 구조 이해하기

### 주요 파티션 구성 및 설명

| Label     | Offset     | Type | Subtype | 설명 |
|-----------|------------|------|---------|------|
| `nvs`     | `0x9000`   | DATA | WIFI    | 비휘발성 저장소 (Wi-Fi 설정, 키 등) |
| `otadata` | `0xe000`   | DATA | OTA     | OTA 관련 메타데이터 |
| `app0`    | `0x10000`  | APP  | ota_0   | 메인 애플리케이션 영역 |
| `app1`    | `0x150000` | APP  | ota_1   | OTA용 대체 앱 영역 |
| `spiffs`  | `0x290000` | DATA | unknown | SPIFFS 파일 시스템 |
| `coredump`| `0x3f0000` | DATA | unknown | 시스템 크래시 덤프 저장 영역 |

이 구조는 [esp_image_parser](https://github.com/tenable/esp32_image_parser)를 통해 간편하게 확인할 수 있어요:

```bash
python3 esp32_image_parser.py show_partitions flash.bin
reading partition table...
entry 0:
  label      : nvs
  offset     : 0x9000
  length     : 20480
  type       : 1 [DATA]
  sub type   : 2 [WIFI]

entry 1:
  label      : otadata
  offset     : 0xe000
  length     : 8192
  type       : 1 [DATA]
  sub type   : 0 [OTA]

entry 2:
  label      : app0
  offset     : 0x10000
  length     : 1310720
  type       : 0 [APP]
  sub type   : 16 [ota_0]

entry 3:
  label      : app1
  offset     : 0x150000
  length     : 1310720
  type       : 0 [APP]
  sub type   : 17 [ota_1]

entry 4:
  label      : spiffs
  offset     : 0x290000
  length     : 1441792
  type       : 1 [DATA]
  sub type   : 130 [unknown]

entry 5:
  label      : coredump
  offset     : 0x3f0000
  length     : 65536
  type       : 1 [DATA]
  sub type   : 3 [unknown]

MD5sum:
972dae2ff872a0142d60bad124c0666b
Done
```

---

## 🔍 app0 영역 추출하기

애플리케이션 코드가 저장된 `app0` 영역은 아래 명령어로 추출할 수 있어요:

```bash
(base) ➜ user@mac  ~/ctf/rev/physiaxl  dd if=flash.bin of=app0.bin bs=1 count=1310720 skip=0x10000
1310720+0 records in
1310720+0 records out
1310720 bytes transferred in 1.829849 secs (716300 bytes/sec)
(base) ➜ user@mac  ~/ctf/rev/physiaxl  file app0.bin
app0.bin: DOS executable (COM)
(base) ➜ user@mac  ~/ctf/rev/physiaxl  hexdump -C app0.bin | head
00000000  e9 05 02 2f 8c 27 08 40  ee 00 00 00 00 00 00 00  |.../.'.@........|
00000010  00 ff ff 00 00 00 00 01  20 00 40 3f 18 2b 01 00  |........ .@?.+..|
00000020  32 54 cd ab 00 00 00 00  00 00 00 00 00 00 00 00  |2T..............|
00000030  33 37 64 36 39 63 61 00  00 00 00 00 00 00 00 00  |37d69ca.........|
00000040  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
00000050  61 72 64 75 69 6e 6f 2d  6c 69 62 2d 62 75 69 6c  |arduino-lib-buil|
00000060  64 65 72 00 00 00 00 00  00 00 00 00 00 00 00 00  |der.............|
00000070  31 32 3a 31 31 3a 35 36  00 00 00 00 00 00 00 00  |12:11:56........|
00000080  46 65 62 20 31 32 20 32  30 32 35 00 00 00 00 00  |Feb 12 2025.....|
00000090  76 35 2e 33 2e 32 2d 35  38 34 2d 67 34 38 39 64  |v5.3.2-584-g489d|
```

추출된 `app0.bin`은 ESP32용 바이너리 포맷으로, 첫 바이트가 `0xE9`인 걸 통해 확인할 수 있습니다. `hexdump`를 통해 컴파일 정보나 프로젝트 이름, 빌드 시간 등을 엿볼 수 있는데, 이는 이후 분석 과정에서 유용하게 쓰여요.

---

## 💥 coredump 추출 및 분석 준비

시스템 크래시 당시 메모리 상태를 담고 있는 `coredump`는 다음과 같이 추출해요:

```bash
(base) ➜ user@mac  ~/ctf/rev/physical  dd if=flash.bin of=coredump bs=1 count=65536 skip=0x3f0000
65536+0 records in
65536+0 records out
65536 bytes transferred in 0.120672 secs (543092 bytes/sec)
(base) ➜ user@mac  ~/ctf/rev/physical  file coredump
coredump: data
(base) ➜ user@mac  ~/ctf/rev/physical  hexdump -C coredump| head
00000000  c4 29 00 00 02 01 00 00  00 00 00 00 00 00 00 00  |.)..............|
00000010  00 00 00 00 2d 01 00 00  7f 45 4c 46 01 01 01 00  |....-....ELF....|
00000020  00 00 00 00 00 00 00 00  04 00 5e 00 01 00 00 00  |..........^.....|
00000030  00 00 00 00 34 00 00 00  00 00 00 00 00 00 00 00  |....4...........|
00000040  34 00 20 00 10 00 28 00  00 00 00 00 04 00 00 00  |4. ...(.........|
00000050  34 02 00 00 00 00 00 00  00 00 00 00 a0 10 00 00  |4...............|
00000060  a0 10 00 00 06 00 00 00  00 00 00 00 01 00 00 00  |................|
00000070  d4 12 00 00 88 81 fb 3f  88 81 fb 3f 60 01 00 00  |.......?...?`...|
00000080  60 01 00 00 06 00 00 00  00 00 00 00 01 00 00 00  |`...............|
00000090  34 14 00 00 30 21 fb 3f  30 21 fb 3f e0 01 00 00  |4...0!.?0!.?....|
```

해당 파일은 ELF 형식이며, `readelf`를 통해 NOTE 섹션을 분석할 수 있어요:

```bash
root@ubuntu:~/ctf/codegate/rev/physical# readelf -n coredump

Displaying notes found at file offset 0x00000234 with length 0x000010a0:
  Owner                Data size 	Description
  CORE                 0x0000024c	NT_PRSTATUS (prstatus structure)
  CORE                 0x0000024c	NT_PRSTATUS (prstatus structure)
  CORE                 0x0000024c	NT_PRSTATUS (prstatus structure)
  CORE                 0x0000024c	NT_PRSTATUS (prstatus structure)
  CORE                 0x0000024c	NT_PRSTATUS (prstatus structure)
  CORE                 0x0000024c	NT_PRSTATUS (prstatus structure)
  CORE                 0x0000024c	NT_PRSTATUS (prstatus structure)

Displaying notes found at file offset 0x00002874 with length 0x0000011c:
  Owner                Data size 	Description
  ESP_CORE_DUMP_INFO   0x00000048	Unknown note type: (0x0000204a)
   description data:
    02 01 00 00 32 66 39 63 39 66 63 35 66 00 00 00  
    00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  
    00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  
    00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  
  ESP_EXTRA_INFOIN     0x00000098	Unknown note type: (0x000002a5)
   description data:
    88 81 fb 3f e8 00 00 00 1c 00 00 00 ee 00 00 00  
    00 00 00 00 c2 00 00 00 00 00 00 00 c3 00 00 00  
    00 00 00 00 c4 00 00 00 00 00 00 00 c5 00 00 00  
    20 0a 06 00 c6 00 00 00 00 00 00 00 b1 00 00 00  
    73 45 08 40 b2 00 00 00 00 00 00 00 b3 00 00 00  
    00 00 00 00 b4 00 00 00 00 00 00 00 b5 00 00 00  
    49 23 08 40 b6 00 00 00 00 00 00 00 00 00 00 00  
    00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  
    00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  
```

NOTE 섹션에서 확인 가능한 주요 항목:
- `NT_PRSTATUS`: 각 코어의 레지스터 상태
- `ESP_CORE_DUMP_INFO`: 코어 정보 요약
- `ESP_EXTRA_INFOIN`: 예외 발생 시점의 백트레이스 및 기타 정보

이 정보를 활용하면 crash 당시 어떤 함수가 실행 중이었는지, 어떤 상황이었는지를 역추적할 수 있어요.

---

## 🔧 app0.bin 구조 확인하기 (esptool)

`esptool`의 `image-info` 명령어를 사용하면 바이너리 구조를 쉽게 파악할 수 있어요:

```bash
(bk) ➜ user@mac  ~/tools/esptool git:(master) PYTHONPATH=`pwd` python3 -m esptool image-info ~/ctf/rev/physical/app0.bin
esptool.py v4.8.1
Image size: 1310720 bytes
Detected image type: ESP32

ESP32 Image Header
==================
Image version: 1
Entry point: 0x4008278c
Segments: 5
Flash size: 4MB
Flash freq: 80m
Flash mode: DIO

ESP32 Extended Image Header
===========================
WP pin: 0xee (disabled)
Flash pins drive settings: clk_drv: 0x0, q_drv: 0x0, d_drv: 0x0, cs0_drv: 0x0, hd_drv: 0x0, wp_drv: 0x0
Chip ID: 0 (ESP32)
Minimal chip revision: v0.0, (legacy min_rev = 0)
Maximal chip revision: v655.35

Segments Information
====================
Segment   Length   Load addr   File offs  Memory types
-------  -------  ----------  ----------  ------------
      0  0x12b18  0x3f400020  0x00000018  DROM
      1  0x03fd8  0x3ffbdb60  0x00012b38  BYTE_ACCESSIBLE, DRAM
      2  0x094f8  0x40080000  0x00016b18  IRAM
      3  0x2bcdc  0x400d0020  0x00020018  IROM
      4  0x062fc  0x400894f8  0x0004bcfc  IRAM

ESP32 Image Footer
==================
Checksum: 0xbf (valid)
Validation hash: 4280fcb5073d4f6cd1b51aeae5268bbb9b5f39e1223e59670ed4db68eb20fbff (valid)

Application Information
=======================
Project name: arduino-lib-builder
App version: 37d69ca
Compile time: Feb 12 2025 12:11:56
ELF file SHA256: 00d64733c51c91d7a571d650044532435cf2aea1a77ac0d1279e7fce2389d181
ESP-IDF: v5.3.2-584-g489d7a2b3a-dirty
Minimal eFuse block revision: 0.0
Maximal eFuse block revision: 0.99
Secure version: 0
```

### 출력 요약
- Entry Point: `0x4008278c`
- Segment 수: 5개

| Segment | Load Addr   | File Offset | Length   | Type         |
|---------|-------------|-------------|----------|--------------|
| 0       | 0x3F400020  | 0x00000018  | 0x12b18  | DROM         |
| 1       | 0x3FFBDB60  | 0x00012B38  | 0x03fd8  | DRAM         |
| 2       | 0x40080000  | 0x00016B18  | 0x094f8  | IRAM         |
| 3       | 0x400D0020  | 0x00020018  | 0x2bcdc  | IROM (.text) |
| 4       | 0x400894F8  | 0x0004BCFC  | 0x062fc  | IRAM         |

추가로 확인 가능한 정보:
- 프로젝트 이름: `arduino-lib-builder`
- ESP-IDF 버전: `v5.3.2`
- 빌드 시간: `2025-02-12 12:11:56`

---

## 🧠 Ghidra로 본격 분석 시작하기

### 📍 메모리 매핑 설정

app0.bin은 ELF 포맷이 아니기 때문에 Ghidra에 Raw Binary로 불러온 후 수동으로 메모리 매핑을 지정해줘야 해요. 

#### 설정 방법
- Language: `Xtensa:LE:32:default`
- Segment 매핑 예시:

```text
0x00000018 → 0x3F400020 (DROM)
0x00012B38 → 0x3FFBDB60 (DRAM)
0x00016B18 → 0x40080000 (IRAM)
0x00020018 → 0x400D0020 (IROM - .text)
0x0004BCFC → 0x400894F8 (IRAM)
```

이 과정을 정확히 진행하지 않으면 Entry Point 분석이 흐트러지거나, 함수 위치가 엉뚱하게 잡힐 수 있으니 주의가 필요해요.

### 🏁 Entry Point에서 main 추적하기

Entry Point로 지정된 `0x4008278c`에서 분석을 시작하면 `FUN_40082794` 함수에 도달하게 돼요. 이 함수는 초기화 루틴을 담당하며, 그 안에서 `FUN_400826b0` 같은 하위 함수들을 호출합니다.

`FUN_400826b0`는 IO 설정, GPIO 초기화 등을 진행하는 코드로 보이며, 전체적인 흐름을 보면 `FUN_40082794`가 사실상 main 함수 역할을 한다고 판단할 수 있어요.

분석 흐름 요약:
1. 코어 리셋 여부 및 초기 상태 점검
2. 주변 장치 초기화
3. 백트레이스 설정 및 예외 핸들링 초기화
4. 사용자 로직 진입을 위한 준비

---

## 📌 분석 포인트 및 팁

다음과 같은 전략을 참고하면 보다 효율적으로 분석할 수 있어요:

- `xref`, `call graph`, `string reference` 기능 적극 활용
- `ESP_EXTRA_INFOIN` 내 백트레이스 주소와 코드상의 함수 매칭
- `app0.bin`과 `coredump`의 메모리 매핑을 기반으로 실행 지점 재현
- `espcoredump.py` + `xtensa-esp32-elf-gdb`를 통해 실시간 디버깅 구현

---

## 🔜 끝으로

이 문제의 직접적인 풀이는 해당 포스팅에 없습니다. 왜냐구요? 못 풀었어요 ㅎ...
혹시라도 푼다면 추가 write-up을 올려보도록 할게요. 하지만 안 올릴 확률이 더 높아요.
이만 여기까지 글을 쓰도록 하겠습니다! 👋



