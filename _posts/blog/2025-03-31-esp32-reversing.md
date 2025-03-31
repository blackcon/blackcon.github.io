---
title: CTF 문제로 알아보는 ESP32 리버싱
categories: [Hacking, CTF]
tags: [reversing, esp32, hardware]
date: 2025-03-31 20:00:00 +0900
---

Codgate 2025에 나온 리버싱 문제 `physical` 바이너리를 분석해봅니다.
blah
작성중...

# flash.bin 파티션 정보
## 파티션 별 역할 알아보기
- nvs: 비휘발성 저장소, WiFi 설정 등을 저장
- otadata: OTA(Over-The-Air) 업데이트 관련 데이터
- app0/app1: 메인 애플리케이션 코드(OTA 업데이트를 위해 두 개 존재)
- spiffs: 파일 시스템
- coredump: 크래시 발생 시 디버깅 정보 저장

## 문제 바이너리의 파니션 정보 확인하기
- 사용한 툴: [esp_image_parser](https://github.com/tenable/esp32_image_parser/blob/master/esp32_image_parser.py)
- 명령어 및 결과
  ```bash
  (bk) ➜ user@mac  ~/tools/esp32_image_parser git:(master) ✗ python3 esp32_image_parser.py show_partitions ~/ctf/rev/physical/flash.bin
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

# app0 label 추출
- 명령어: dd if=`input_file` of=`output_file` bs=1 count=`length` skip=`offset`
- 명령어 실행 및 결과
  ```bash
  (base) ➜ user@mac  ~/ctf/rev/physical  dd if=flash.bin of=app0.bin bs=1 count=1310720 skip=0x10000
  1310720+0 records in
  1310720+0 records out
  1310720 bytes transferred in 1.829849 secs (716300 bytes/sec)
  (base) ➜ user@mac  ~/ctf/rev/physical  file app0.bin
  app0.bin: DOS executable (COM)
  (base) ➜ user@mac  ~/ctf/rev/physical  hexdump -C app0.bin | head
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
  > bin 파일의 첫 바이트가 `e9`인 것은 esp binary를 뜻함

# coredump 추출
- 명령어: dd if=`input_file` of=`output_file` bs=1 count=`length` skip=`offset`
- 명령어 실행 및 결과
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

# coredump 파일의 notes 확인
- 추출한 coredump에서 최소한의 정보를 얻어보고자`note`를 출력했다. (하지만 얻을만한 것은 없었음)
- 명령어 실행 및 결과
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
  
# app0.bin의 image-info
- 앞서 추출한 `app0.bin` 이미지의 정보를 보기위해 [esptool](https://github.com/espressif/esptool) 도구를 활용하여 정보를 출력한다.
- 획득 가능한 정보: `Entry pont`, `Segments Information`
- 명령어 실행 및 결과
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

# Ghidra로 열어보기
## memory mapping하기
## Entry Point 추적
- app0.bin의 image-info에서 획득한 Entry point를 쫓아가면, main으로 보여지진 않지만 어느 함수가 존재한다. 아마 crash당시의 entry 이려나..?
  ```c
  void FUN_400826b0(void)
  
  {
    uint *puVar1;
    uint uVar2;
    uint uVar3;
    uint uVar4;
    uint uVar5;
    
    memw();
    if ((*(uint *)(DAT_400805a8 + 0xc) & 1) == 0) {
      FUN_40082b28();
      (*DAT_400804e4)(1);
      FUN_40082b84();
      FUN_40082b28();
      (*DAT_400805b0)(1);
      FUN_40082b84();
      puVar1 = DAT_40080534;
      uVar2 = FUN_40083a40(DAT_40080534);
      memw();
      *puVar1 = uVar2 | 0x2000;
      (*DAT_400805b4)(1);
      uVar2 = FUN_40083a40(puVar1);
      memw();
      *puVar1 = uVar2 & 0xffffdfff;
    }
    memw();
    memw();
    DAT_400805ac[0x800] = *DAT_400805ac;
    uVar2 = FUN_40083a40(DAT_40080538);
    puVar1 = DAT_40080534;
    uVar4 = (uVar2 >> 2 & 1 ^ 1) << 2 | (uVar2 >> 1 & 1 ^ 1) * 2 | (uVar2 ^ 0xffffffff) & 1;
    uVar5 = (uVar2 >> 3 & 1 ^ 1) << 4;
    uVar3 = FUN_40083a40(DAT_40080534);
    memw();
    *puVar1 = ((uVar5 | (uVar2 >> 4 & 1 ^ 1) << 3 | uVar4) * 2 & 0x10 | uVar5 >> 1 | uVar4) & uVar3 ^
              uVar3;
    memw();
    return;
  }
  ```
- 바로 위의 함수인 `FUN_400826b0`를 어디서 호출하는지 추적을 해본다. 딱 1개가 존재하는데, 여기거 실제 main으로 보여진다.
- Disassembly
  ```c
  
  void FUN_40082794(void)
  
  {
    byte bVar1;
    byte bVar2;
    int *piVar3;
    byte *pbVar4;
    int iVar5;
    uint *puVar6;
    int iVar7;
    undefined4 uVar8;
    undefined4 uVar9;
    uint uVar10;
    int iVar11;
    undefined4 uVar12;
    undefined1 in_VECBASE;
    undefined1 in_PRID;
    undefined8 uVar13;
    undefined4 local_50;
    undefined4 uStack_4c;
    undefined4 uStack_48;
    undefined4 uStack_44;
    undefined4 uStack_40;
    undefined4 uStack_3c;
    byte abStack_38 [10];
    byte bStack_2e;
    
    wsr(in_VECBASE,PTR_LOOP_40080578);
    iVar7 = (*DAT_400805fc)(0);
    (*DAT_400805fc)(1);
    FUN_40086064(DAT_400805bc,0,DAT_400805b8 - DAT_400805bc);
    if (iVar7 != 5) {
      FUN_40086064(DAT_400805c4,0,DAT_400805c0 - DAT_400805c4);
    }
    FUN_400826b0();
    func_0x4008bb08();
    FUN_40085460();
    (*DAT_40080600)();
    (*DAT_40080604)();
    FUN_40085468();
    (*DAT_40080608)();
    (*DAT_4008060c)();
    (*DAT_40080594)();
    pbVar4 = DAT_4008057c;
    memw();
    *DAT_4008057c = 1;
    (*DAT_40080610)(abStack_38);
    piVar3 = DAT_40080474;
    if (bStack_2e < 2) {
      if (*DAT_40080474 == 0) goto LAB_40082845;
      uVar8 = func_0x4008e090();
      uVar12 = DAT_400805c8;
      (*(code *)PTR_SUB_40080480)(DAT_400805cc,uVar8,DAT_400805c8);
      if (*piVar3 == 0) goto LAB_40082845;
      uVar9 = func_0x4008e090();
      uVar8 = DAT_400805d0;
    }
    else {
      FUN_40082b28();
      (*DAT_400805b0)(1);
      FUN_40082b84();
      FUN_40082b28();
      (*DAT_400804e0)(1);
      FUN_40082b84();
      FUN_40086afc(1);
      puVar6 = DAT_400805f8;
      uVar10 = FUN_40083a40(DAT_400805f8);
      if ((uVar10 & 1) == 0) {
        uVar10 = FUN_40083a40(puVar6);
        memw();
        *puVar6 = uVar10 | 1;
        puVar6 = DAT_400805f0;
        uVar10 = FUN_40083a40(DAT_400805f0);
        memw();
        *puVar6 = uVar10 & 0xfffffffe;
        puVar6 = DAT_400805f4;
        uVar10 = FUN_40083a40(DAT_400805f4);
        memw();
        *puVar6 = uVar10 | 1;
        uVar10 = FUN_40083a40(puVar6);
        memw();
        *puVar6 = uVar10 & 0xfffffffe;
      }
      (*DAT_40080590)(PTR_LAB_400805d4);
      do {
        memw();
        bVar1 = *pbVar4;
        memw();
        bVar2 = pbVar4[1];
        (*DAT_400804d4)(100);
      } while ((bVar2 & bVar1) == 0);
      (*DAT_40080614)();
      (*DAT_40080618)();
      uVar13 = (*DAT_4008061c)();
      *DAT_400805d8 = uVar13;
      (*DAT_400805a0)();
      iVar11 = FUN_40083a74();
      FUN_400834f4(0);
      iVar5 = DAT_400805dc;
      uVar10 = (uint)((ulonglong)(uint)(iVar11 << 4) * (ulonglong)DAT_400805e8 >> 0x20);
      memw();
      memw();
      *(uint *)(DAT_400805dc + 0x14) = *(uint *)(DAT_400805dc + 0x14) & 0xfff00000 | uVar10 >> 0x14;
      memw();
      memw();
      *(uint *)(iVar5 + 0x14) =
           *(uint *)(iVar5 + 0x14) & DAT_400805ec | (uVar10 >> 0x10 & 0xf) << 0x14;
      if (iVar7 == 5) {
        (*DAT_40080620)();
      }
      (*DAT_400805a4)();
      local_50 = 0;
      uStack_4c = 0;
      uStack_48 = 0;
      uStack_44 = 0;
      uStack_40 = 0;
      uStack_3c = 0;
      FUN_40085f00(&local_50,DAT_400805e4,0x18);
      if ((char)local_50 == -0x17) {
        FUN_40084b3c(&local_50);
        FUN_40084b74(&local_50);
        FUN_40084d48(&local_50);
        FUN_40084ac8();
        pbVar4 = DAT_40080580;
        memw();
        *DAT_40080580 = 1;
        memw();
        abStack_38[0] = 0;
        while (memw(), abStack_38[0] == 0) {
          memw();
          memw();
          memw();
          memw();
          memw();
          memw();
          abStack_38[0] = *pbVar4 & 1 & pbVar4[1];
          memw();
          (*DAT_400804d4)(100);
        }
        uVar10 = rsr(in_PRID);
        (**(code **)((uVar10 >> 0xd & 1) * 4 + DAT_40080588))();
        memw();
        return;
      }
      if (*DAT_40080474 == 0) goto LAB_40082845;
      uVar9 = func_0x4008e090();
      uVar8 = DAT_400805e0;
      uVar12 = DAT_400805c8;
    }
    (*(code *)PTR_SUB_40080480)(uVar8,uVar9,uVar12);
  LAB_40082845:
    func_0x4008e0d4();
    return;
  }
  ```

