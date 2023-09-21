---
title: C와 C++ 코드에서 메모리 오류를 감지하는 도구 | Address Sanitizer 
categories: [Tools, AddressSnitizer]
tags: [WriteUsingLLaMA]
date: 2023-9-21 09:50:0 +0900
---

C와 C++은 강력한 프로그래밍 언어이지만, [메모리 관리에 관한 취약점](https://blackcon.github.io/posts/theory-binary-vuln/)(혹은 버그)가 존재할 수 있습니다. 메모리 오류는 프로그램에서 치명적인 결함을 일으킬 수 있으며, 개발 단계에서 부터 이러한 오류를 미리 찾아내고 예방히기는 여간 어려운게 아니죠. 

이러한 문제를 해결하기 위해 **Google**에서는 [Sanitizers](https://github.com/google/sanitizers)라는 프로젝트를 공개했습니다. 이 [Sanitizers](https://github.com/google/sanitizers)에는 **AddressSanitizer**, **MemorySanitizer**, **HWASAN**, **UBSan** 이런 기능을 각기 제공하고 있으며, 커널 소스코드를 위해서는 **KASAN**, **KMSAN**, **KCSAN**도 함께 제공하고 있습니다.

이번 포스트에서는 [Address Sanitizer (ASan)](https://github.com/google/sanitizers/wiki/AddressSanitizer)를 사용하기 위한 셋팅 방법과 어떻게 사용하는지에 대해 알아보겠습니다.

Address Sanitizer (ASan)
===========================

### 1.1) ASan이란?

ASan은 C 및 C++ 프로그램에서 메모리 오류를 식별하고 디버그하는 데 사용되는 도구입니다. ASan은 주로 다음과 같은 메모리 오류 유형을 검출합니다.

| 메모리 오류 유형                              | 설명                                               |
|--------------------------------------------|----------------------------------------------------|
| [Buffer Overflow](https://blackcon.github.io/posts/theory-bof/)        | 배열 경계를 초과하여 데이터를 읽거나 쓸 때 발생하는 오류를 감지합니다.              |
| **Buffer Underflow**        | 배열 경계를 넘어서 이전에 할당된 메모리 영역을 읽을 때 발생하는 오류를 감지합니다.  |
| [Use-after-Free](https://blackcon.github.io/posts/theory-uaf/)       | 이미 해제된 메모리를 다시 사용하려고 할 때 발생하는 오류를 감지합니다.           |
| [Double Free](https://blackcon.github.io/posts/theory-double-free/)                  | 이미 해제된 메모리를 다시 해제하려고 할 때 발생하는 오류를 감지합니다.          |
| [Heap Buffer Overflow](https://blackcon.github.io/posts/theory-heapoverflow/) | 동적으로 할당된 메모리(heap)의 경계를 초과하여 데이터를 읽거나 쓸 때 발생하는 오류를 감지합니다. |
| [Stack Buffer Overflow](https://blackcon.github.io/posts/theory-bof/) | 스택 메모리 영역의 배열 경계를 초과하여 데이터를 읽거나 쓸 때 발생하는 오류를 감지합니다. |
| [Global Buffer Overflow](https://blackcon.github.io/posts/theory-bof/) | 전역 메모리 영역의 배열 경계를 초과하여 데이터를 읽거나 쓸 때 발생하는 오류를 감지합니다. |
| **Use after return** | 함수에서 반환한 후에 해당 함수 내에서 할당한 메모리를 사용하려고 할 때 발생하는 오류를 감지합니다. |
| **Use after scope** | 지역 변수의 범위를 벗어난 사용 (use-after-scope)을 검출합니다. |
| **Initialization Order Bugs** | 객체 초기화 순서에 따라 발생하는 오류를 검출합니다. |
| **Memory Leaks** | 프로그램이 동적으로 할당한 메모리를 해제하지 않고 종료될 때 발생하는 문제를 감지합니다. |


그럼 이제, ASan을 어떻게 설치하고 사용하는지 알아보겠습니다. ASan을 사용하기 위해서는 기본적으로 **Clang**이 설치되어 있어야 하며, 분석하고자 하는 소스코드를 가지고 있어야 합니다.

### 1.2) Clang 설치 | ASan 환경셋팅
- ASan은 Clang 컴파일러와 함께 제공됩니다. 운영 체제에 따라 Clang를 설치합니다.
- Clang 컴파일러를 설치하는 방법은 운영 체제 및 배포판에 따라 다를 수 있으니, 환경별로 설치할 때 유의해주세요 :)
- 설치 방법
  - **Ubuntu 또는 Debian:**
    1. 터미널을 열고 다음 명령을 실행하여 패키지 목록을 업데이트합니다:

        ******bash
        sudo apt update
        ******

    2. Clang을 설치합니다:
        ******bash
        sudo apt install clang
        ******

    3. 설치가 완료되면 다음 명령으로 설치된 Clang 버전을 확인할 수 있습니다:
        ******bash
        clang --version
        ******

  - **Fedora:**
    1. 터미널을 열고 다음 명령으로 Clang을 설치합니다:
        ******bash
        sudo dnf install clang
        ******

    2. 설치가 완료되면 다음 명령으로 설치된 Clang 버전을 확인할 수 있습니다:
        ******bash
        clang --version
        ******

  - **macOS (Homebrew를 사용하는 경우):**
    1. Homebrew가 설치되어 있지 않다면, 터미널에서 다음 명령으로 Homebrew를 설치합니다:
        ******bash
        /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
        ******

    2. Homebrew를 사용하여 Clang을 설치합니다:
        ******bash
        brew install llvm
        ******

    3. 설치가 완료되면 다음 명령으로 설치된 Clang 버전을 확인할 수 있습니다:
        ******bash
        clang --version
        ******


### 1.3) **ASan** 활성화 및 실행 | ASan 실습
- 기존 리눅스에서는 컴파일을 할 때 **gcc**를 사용했겠지만, ASan을 사용하기 위해서는 앞서 설치한 **Clang** 을 사용하여 컴파일 합니다.
- 그리고 **Clang**으로 컴파일할 때 **-fsanitize=address** 플래그를 사용하여 ASan을 활성화할 수 있습니다.
- **예제 (1)**
   - 소스코드
        ******c
        #include <stdio.h>
        #include <stdlib.h>
        #include <string.h>

        int main()
        {
            char *buf = NULL;

            buf = (char *)malloc(100);
            memset((void *)buf, 0x61, sizeof(buf)+1);

            return 0;
        }
        ******
    - 컴파일 및 실행
        ******
        ➜  ~ clang -fsanitize=address -o test ./test.c
        ➜  ~ ./test

        =================================================================
        ==2018==ERROR: LeakSanitizer: detected memory leaks

        Direct leak of 100 byte(s) in 1 object(s) allocated from:
            #0 0x4c3b78  (/home1/irteamsu/temp/test+0x4c3b78) (BuildId: 379bf4d826c30440d02360d752a7833a948020e0)
            #1 0x512f10  (/home1/irteamsu/temp/test+0x512f10) (BuildId: 379bf4d826c30440d02360d752a7833a948020e0)
            #2 0x7fbfb8041d84  (/lib64/libc.so.6+0x3ad84) (BuildId: 31f2a86084da882dfe4ecc1fe2a9eca8ce9416fd)

        SUMMARY: AddressSanitizer: 100 byte(s) leaked in 1 allocation(s). 
        ******

- **예제 (2)**
   - 소스코드
        ******bash
        ➜  ~ cat test.c
        #include <stdio.h>
        #include <stdlib.h>
        #include <string.h>

        int main()
        {
            char *buf = NULL;

            buf = (char *)malloc(100);
            memset((void *)buf, 0x61, sizeof(buf)+1);

            return 0;
        }
        ******

    - 컴파일 및 실행
        ******
        ➜  ~ clang -fsanitize=address -o test ./test.c
        ➜  ~ ./test
        =================================================================
        ==26375==ERROR: AddressSanitizer: stack-buffer-overflow on address 0x7f2a4480002a at pc 0x00000051301b bp 0x7fffe1c77cd0 sp 0x7fffe1c77cc0
        WRITE of size 1 at 0x7f2a4480002a thread T0
            #0 0x51301a  (/home1/irteamsu/temp/test+0x51301a) (BuildId: fdb1297446db5dae328c9d5c2cd9e59eac69ce86)
            #1 0x7f2a46e97d84  (/lib64/libc.so.6+0x3ad84) (BuildId: 31f2a86084da882dfe4ecc1fe2a9eca8ce9416fd)
            #2 0x41f1ad  (/home1/irteamsu/temp/test+0x41f1ad) (BuildId: fdb1297446db5dae328c9d5c2cd9e59eac69ce86)

        Address 0x7f2a4480002a is located in stack of thread T0 at offset 42 in frame
            #0 0x512eff  (/home1/irteamsu/temp/test+0x512eff) (BuildId: fdb1297446db5dae328c9d5c2cd9e59eac69ce86)

        This frame has 1 object(s):
            [32, 42) 'buf' <== Memory access at offset 42 overflows this variable
        HINT: this may be a false positive if your program uses some custom stack unwind mechanism, swapcontext or vfork
            (longjmp and C++ exceptions *are* supported)
        SUMMARY: AddressSanitizer: stack-buffer-overflow (/home1/irteamsu/temp/test+0x51301a) (BuildId: fdb1297446db5dae328c9d5c2cd9e59eac69ce86)
        Shadow bytes around the buggy address:
        0x0fe5c88f7fb0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
        0x0fe5c88f7fc0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
        0x0fe5c88f7fd0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
        0x0fe5c88f7fe0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
        0x0fe5c88f7ff0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
        =>0x0fe5c88f8000: f1 f1 f1 f1 00[02]f3 f3 00 00 00 00 00 00 00 00
        0x0fe5c88f8010: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
        0x0fe5c88f8020: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
        0x0fe5c88f8030: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
        0x0fe5c88f8040: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
        0x0fe5c88f8050: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
        Shadow byte legend (one shadow byte represents 8 application bytes):
        Addressable:           00
        Partially addressable: 01 02 03 04 05 06 07
        Heap left redzone:       fa
        Freed heap region:       fd
        Stack left redzone:      f1
        Stack mid redzone:       f2
        Stack right redzone:     f3
        Stack after return:      f5
        Stack use after scope:   f8
        Global redzone:          f9
        Global init order:       f6
        Poisoned by user:        f7
        Container overflow:      fc
        Array cookie:            ac
        Intra object redzone:    bb
        ASan internal:           fe
        Left alloca redzone:     ca
        Right alloca redzone:    cb
        ==26375==ABORTING
        ******

## 마무리

ASan은 C 및 C++ 프로그램에서 발생할 수 있는 메모리 오류를 검출하고 디버그하는 데 강력한 도구입니다. 이러한 도구를 활용하여 프로그램의 안정성을 향상시키고 메모리 오류로 인한 버그를 방지할 수 있습니다. ASan과 MSan은 개발자들에게 신뢰성 있는 코드를 작성하는 데 큰 도움이 됩니다.

이제 ASan을 사용하여 프로그램의 메모리 오류를 신속하게 검출하고 해결할 수 있을 것입니다. Happy coding!