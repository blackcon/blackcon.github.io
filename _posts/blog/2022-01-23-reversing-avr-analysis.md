---
title: QEMU 를 이용한 AVR 바이너리 분석하는 방법
categories: [Hacking, Reversing]
tags: [Hacking, Reversing]
date: 2022-01-23 23:08:00 +0900
---
## 1. 환경
-   host: macOS
-   target: ELF 32-bit LSB executable, Atmel AVR 8-bit,

## 2. QEMU 설치
-   brew 를 사용한 install
    >   brew install qemue
-   apt 를 사용한 install
    >   apt install qemu

## 3. AVR 바이너리(ELF) 파일 실행
-   참고  
    [https://qemu.readthedocs.io/en/latest/system/target-avr.html](https://qemu.readthedocs.io/en/latest/system/target-avr.html)
-   예제 
    ```bash
    qemu-system-avr \
        -machine arduino-duemilanove \
        -bios `atme.elf` -s -S
    ```

## 4. avr-gdb 설치

-   First, make sure you have xcode command line developer tools installed with  
    ```bash
    $ xcode-select --install
    ```
-   Then, just run the following to install the latest version of avr-gcc:  
    ```bash
    $ brew tap osx-cross/avr  
    $ brew install avr-gcc  
    $ brew install avr-gdb
    ```

## 5. avr-gdb 를 이용한 디버깅 실행

-   avr-gdb 명령어 실행
    ```bash
    user@bk-mac:~/ctf $ avr-gdb ./atme.elf 

    GNU gdb (GDB) 10.1 Copyright (C) 2020 Free Software Foundation, Inc. 
    License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html> 
    This is free software: you are free to change and redistribute it. There is NO WARRANTY, to the extent permitted by law. Type "show copying" and "show warranty" for details. This GDB was configured as "--host=x86_64-apple-darwin20.6.0 --target=avr". Type "show configuration" for configuration details. 
    For bug reporting instructions, please see: <https://www.gnu.org/software/gdb/bugs/>. 
    Find the GDB manual and other documentation resources online at: <http://www.gnu.org/software/gdb/documentation/>. 
    For help, type "help". Type "apropos word" to search for commands related to "word"...
    ```
    
-  qemu에 실행되어 있는 ELF 파일에 원격 연결
    ```bash
    (gdb) target remote :1234 
    Remote debugging using :1234 
    warning: Target-supplied registers are not supported by the current architecture 0x00000000 in __vectors ()
    ```
- 디버깅 예제  
    ![](https://blog.kakaocdn.net/dn/X0qzV/btrrpU1MNAn/QcgS7clzsWKtJojD6ME7p0/img.png)
