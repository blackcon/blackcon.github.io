---
title: ROP (Return Oriented Programming) 란?
categories: [Hacking, Theory]
tags: [Hacking, System Hacking]
date: 2024-12-21 01:40:0 +0900
---

### 목차

**1장. ROP 개요**  
1.1 ROP란 무엇인가?  
1.2 기존 메모리 손상 공격 기법과의 차이점  
1.3 현대 보안 기법(NX, ASLR)과 ROP의 등장 배경

**2장. ROP의 핵심 개념**  
2.1 ROP와 가젯(Gadget)  
2.2 가젯의 종류와 역할 (레지스터 조작, 메모리 접근, 시스템 콜 트리거)  
2.3 ROP 체인(ROP Chain)의 구조

**3장. ROP의 동작 원리와 체인 구축 과정**  
3.1 스택 프레임 변조와 반환 주소 재설정  
3.2 가젯 체인 연결 기법  
3.3 ROP를 활용한 공격 시나리오 사례 5가지  
   - execve("/bin/sh") 호출  
   - mprotect()로 메모리 권한 변경 후 쉘코드 실행  
   - GOT 오버라이트를 통한 제어 흐름 탈취  
   - 정보 누출(Information Leak)을 통한 ASLR 우회  
   - 스택 피봇(Stack Pivot)을 통한 제어권 재정렬

**4장. x86 환경에서의 ROP 체인 예시**  
4.1 전통적인 x86 cdecl 호출 규약 리뷰  
4.2 strcpy를 통한 문자열 복사 후 system 호출 예제  
4.3 페이로드 예시(Payload) 해설

**5장. x64 환경에서의 ROP 체인 예시**  
5.1 Linux x86_64 ABI 호출 규약 복습 (RDI, RSI, RDX, ...)  
5.2 x64 환경에서 strcpy + system 호출 체인 구성  
5.3 예제 페이로드와 가젯(pop rdi; ret, pop rsi; ret) 활용

**6장. ROP 방어 기법과 대응 전략**  
6.1 Control Flow Integrity(CFI)와 CET 기술  
6.2 강화된 ASLR  
6.3 Stack Canary, 포티파이(Fortify)  
6.4 종합적 방어 전략: DEP + ASLR + CFI 연계

**부록**  
- 참고 자료 및 추천 학습 경로  
- 실습 환경 구축 가이드  
- 추가적인 ROP 툴 및 프레임워크 소개

------------------------------------------------------------

### 1장. ROP 개요

**1.1 ROP란 무엇인가?**  
ROP(Return Oriented Programming)는 현대 보안 환경에서 종종 사용되는 익스플로잇 기법이다. 전통적인 스택 버퍼 오버플로우 공격은 스택 영역에 쉘코드를 삽입하고 `ret`로 점프하여 쉘코드를 실행하는 방식이었다. 그러나 NX(Non-eXecutable) 스택을 적용하면 스택에 삽입된 쉘코드가 실행될 수 없다. ASLR(Address Space Layout Randomization)로 인해 코드의 주소 또한 예측하기 어려워졌다. 이런 방어 기법들이 강화되면서 공격자들은 이미 실행 가능한 코드 세그먼트의 일부를 재활용하기 시작했는데, 이것이 바로 ROP 기법이다.

**1.2 기존 메모리 손상 공격 기법과의 차이점**  
과거에는 단순히 EIP(또는 RIP)를 스택의 쉘코드 주소로 덮어씌워 코드를 실행할 수 있었다. 하지만 DEP/NX로 인해 스택 실행이 불가해지자, 공격자들은 프로그램 내 기 배포된 코드 조각(가젯)을 이어붙여 원하는 동작을 수행하게 되었다. 즉, 새로운 코드를 주입하지 않고 "반환 지향"적으로 기존 코드 블록을 재활용하는 점이 특징이다.

**1.3 현대 보안 기법(NX, ASLR)과 ROP의 등장 배경**  
NX 비트는 스택 메모리를 실행 불가능하게 만듦으로써 전통적인 쉘코드 실행을 차단한다. ASLR은 라이브러리와 실행 파일의 로드 주소를 매번 바꿈으로써 가젯 주소 예측을 어렵게 한다. ROP는 이러한 방어 기법을 우회하기 위해, 임의코드 실행 없이 이미 메모리에 로드된 코드 조각을 체인으로 엮어 원하는 로직을 수행한다.

------------------------------------------------------------

### 2장. ROP의 핵심 개념

**2.1 ROP와 가젯(Gadget)**  
가젯은 바이너리 내에 존재하는 짧은 코드 조각으로, 일반적으로 레지스터 조작이나 메모리 접근, `ret` 명령어로 끝나는 형태를 가진다. 한두 개의 어셈블리 명령어로 구성되는 경우가 많다.

**2.2 가젯의 종류와 역할**  
- 레지스터 로드: `pop rdi; ret` 같은 가젯은 RDI에 특정 값을 로드하는 데 사용  
- 메모리 쓰기/읽기: `mov [rdi], rax; ret` 형태로 메모리 수정  
- syscall 트리거: `syscall; ret` 가젯을 통해 커널 호출  
이들을 적절히 연결하면, 쉘 획득(execve), 메모리 권한 변경(mprotect), 함수 오버라이드 등을 수행할 수 있다.

**2.3 ROP 체인(ROP Chain)의 구조**  
ROP 체인은 스택에 일련의 반환 주소들을 배치하여, 함수 반환 시 마다 가젯을 하나씩 실행하도록 하는 방식이다. 이를 통해 공격자는 사실상 고급 스크립팅 언어처럼 가젯들을 "프로그래밍" 할 수 있다.

------------------------------------------------------------

### 3장. ROP의 동작 원리와 체인 구축 과정

**3.1 스택 프레임 변조와 반환 주소 재설정**  
취약점을 이용해 스택의 RET 주소를 덮어씌운다. 이후 반환 시 EIP/RIP가 우리가 원하는 가젯으로 점프하도록 유도한다.

**3.2 가젯 체인 연결 기법**  
각 가젯은 `ret` 명령어로 끝나므로, 한 가젯을 끝내면 스택에서 다음 주소를 pop하여 다음 가젯으로 이동한다. 이 과정을 반복하면 원하는 로직 구현이 가능하다.

**3.3 ROP를 활용한 공격 시나리오 사례 5가지**  
- **execve("/bin/sh")**: 레지스터에 인자를 셋업하고 syscall 혹은 system 함수를 호출하여 쉘 획득.  
- **mprotect()**: 메모리 영역에 실행 권한을 부여한 뒤 스택/힙 쉘코드를 실행.  
- **GOT 오버라이트**: GOT 테이블 엔트리를 덮어써 호출되는 함수를 공격자 의도대로 변경.  
- **정보 누출**: write/puts 함수를 가젯으로 호출해 메모리 주소를 유출, ASLR 우회.  
- **스택 피봇**: RSP를 다른 메모리 영역으로 옮겨, 더 풍부한 ROP 체인 구성 가능.

------------------------------------------------------------

### 4장. x86 환경에서의 ROP 체인 예시

**4.1 x86 cdecl 호출 규약 리뷰**  
x86에서의 C 함수 호출은 스택에 인자를 푸시하고 `call`을 수행한다. 반환 후에는 호출자가 스택을 정리한다.

**4.2 strcpy를 통한 문자열 복사 후 system 호출 예제**  
- 시나리오:  
  1. strcpy@plt를 호출해 "/bin/sh"를 특정 메모리로 복사  
  2. system@plt를 이용해 복사된 "/bin/sh" 실행
- 공격 절차:  
  스택에 strcpy 호출에 필요한 인자 순서(RET addr, dest, src)를 두고, strcpy 반환 후 system 호출로 이어진다.
  
**4.3 페이로드 예시(Payload) 해설**  
```  
[버퍼 + 패딩]
[strcpy@plt 주소]
[pop pop ret 가젯] # x86에서는 strcpy 종료 후 스택 정리에 필요
[dest 주소]
[src 주소("/bin/sh")]
[system@plt 주소]
[exit@plt 주소]
[dest 주소("/bin/sh" 복사본)]
```
이런 순서로 스택을 구성하면 프로그램 실행 중 strcpy로 문자열을 복사한 뒤, 이어서 system("/bin/sh") 호출에 성공하여 쉘을 획득한다.

------------------------------------------------------------

### 5장. x64 환경에서의 ROP 체인 예시

**5.1 Linux x86_64 ABI 호출 규약 복습**  
x86_64에서는 RDI, RSI, RDX, RCX, R8, R9 레지스터 순으로 함수 인자를 전달한다. 스택에 인자를 푸시할 필요가 줄어들었다.

**5.2 x64 환경에서 strcpy + system 호출 체인 구성**  
- 시나리오:  
  1. `pop rdi; ret`, `pop rsi; ret` 가젯으로 RDI=dest, RSI=src 설정 후 strcpy@plt 호출  
  2. strcpy 후 다시 `pop rdi; ret`로 RDI에 "/bin/sh" 주소를 로드  
  3. system@plt 호출로 쉘 획득

**5.3 예제 페이로드**  
```  
[버퍼 + 패딩]
[pop rdi; ret]
[dest 주소]
[pop rsi; ret]
[src 주소("/bin/sh")]
[strcpy@plt]
[pop rdi; ret]
[dest 주소("/bin/sh" 복사본)]
[system@plt]
[exit@plt]
```
이렇게 하면 x64 환경에서도 동일한 논리로 ROP 체인을 구축할 수 있다.

------------------------------------------------------------

### 6장. ROP 방어 기법과 대응 전략

**6.1 Control Flow Integrity(CFI)와 CET 기술**  
제어 흐름 무결성 보장은 비정상적인 반환 흐름을 감지하고 차단한다. CET(Control-Flow Enforcement Technology)는 하드웨어 레벨에서 ROP/JOP 공격을 방어한다.

**6.2 강화된 ASLR**  
ASLR을 더 빈번히 재배치하거나 PIE(Position Independent Executable)를 활용하면 가젯 주소를 찾기 어렵게 만든다.

**6.3 Stack Canary, Fortify**  
스택 상의 변조를 탐지하기 위해 stack canary를 사용한다. Fortify Source로 함수 호출을 강화하여 버퍼 오버플로우를 사전에 차단한다.

**6.4 종합적 방어 전략: DEP + ASLR + CFI 연계**  
DEP로 스택 실행을 막고, ASLR로 주소를 무작위화하며, CFI로 제어 흐름을 감시하면 ROP 공격 난이도를 크게 높일 수 있다.

------------------------------------------------------------

### 부록

- **참고 자료 및 추천 학습 경로**:  
  - "Hacking: The Art of Exploitation"  
  - "Phrack" 저널, ROP 관련 기사들  
  - 온라인 ROP 챌린지(CTF) 풀이 자료

- **실습 환경 구축 가이드**:  
  - VirtualBox나 Docker로 안전한 실습 환경 조성  
  - gdb, pwndbg, gef 등의 디버거 툴 사용  
  - radare2, rop gadget finder 툴 활용

- **추가적인 ROP 툴 및 프레임워크 소개**:  
  - ROPgadget, Ropper, pwntools 등  
  - 다양한 바이너리에서 손쉽게 가젯을 추출하고 페이로드 제작 지원

------------------------------------------------------------

### 마치며

이 전자책에서는 ROP의 기초 개념부터 시작해 x86, x64 환경에서의 예제, 그리고 방어 기법까지 개략적으로 살펴보았다. 한 달 정도 시스템 해킹을 공부한 독자라면, 이제 ROP의 개념을 이해하고 간단한 체인을 직접 만들어볼 만한 기반을 닦은 셈이다. 이후 더 고난도 환경, 다양한 바이너리와 OS 환경에서 스스로 ROP 체인을 탐색하고 제작해보며 실력을 키워 나가기를 바란다.
