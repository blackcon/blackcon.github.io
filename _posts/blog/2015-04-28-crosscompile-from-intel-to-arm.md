---
title: ARM 어셈블리어 Cross compile하기
categories: [Hacking, Reversing]
tags: [Hacking, Reversing]
date: 2015-04-28 18:05:00 +0900
---
일반 `intel 아키텍처`에서 `ARM 아키텍처`의 프로그램을 생성하는 크로스컴파일(Cross-Compile)을 간략히 설명한다.
테스트 환경은 Ubuntu 14.04 64bit 이다.

> $ uname -a  
> Linux bk 3.13.0-51-generic #84-Ubuntu SMP Wed Apr 15 12:08:34 UTC 2015 x86\_64 x86\_64 x86\_64 GNU/Linux  

그럼 크로스컴파일을 하기전에 apt-get install을 이용하여 아래의 파일들을 다운받는다.  

\[+\] 설치 할 항목

> sudo apt-get install qemu-user libc6-armel-cross libc6-dev-armel-cross binutils-arm-linux-gnueabi gcc-arm-linux-gnueabi

그리고 간단한 어셈코드를 코딩할건데 여기서는 shell을 실행할 수 있는 코드를 작성한다.

\[+\] test.asm

>   1 .text  
>   2 .globl \_start  
>   3 \_start:  
>   4     mov %r7, $11  
>   5     ldr %r0, =msg  
>   6     mov %r1, $0  
>   7     mov %r2, $0  
>   8     svc 0  
>   9 msg:  
>  10     .asciz "/bin/sh"

그 후 컴파일을 하기 위해서 아래와 같은 명령어를 입력한다.

\[+\] Cross Compile

> arm-linux-gnueabi-gcc -o arm.exe -nostartfiles -static test.s

\[+\] ARM 파일 실행하기

> qemu-arm ./arm.exe

\[!\] 실행 결과

[##_Image|t/cfile@21175A40553F2D4B08|CDM|1.3|{"originWidth":640,"originHeight":124,"style":"alignCenter","width":640,"height":124}_##]