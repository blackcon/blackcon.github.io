---
title: 퍼너블/리버싱을 위한 도구 설치 방법 | gdb-peda 설치 가이드
categories: [Tools, gdb]
tags: [gdb, peda, reversing]
date: 2023-09-19 22:00:00 +0900
---
# 1. gdb-peda 란?
- 본 프로젝트는 python으로 개발되어 있으며, `gdb` 사용을 보다 편리하게 사용하기 위한 플러그인이다.
- 보통 CTF 에서 많이 사용되지만, 요즘은 `real-world`의 binary를 분석할 때도 많이 활용하고 있다.

# 2. gdb-peda 설치 방법
- gdb-peda는 [longld](https://github.com/longld)라는 사용자가 개발했으며, 해당 [github](https://github.com/longld/peda)을 통해서 다운받을 수 있다.
- Debian 계열 (kali, ubuntu, ...)
    ```bash
    sudo apt install gdb-peda
    ```
- 혹은 `git clone`으로 설치하기
    ```bash
    git clone https://github.com/longld/peda.git ~/peda
    echo "source ~/peda/peda.py" >> ~/.gdbinit
    echo "DONE! debug your program with gdb and enjoy"
    ```
- gdb-peda 실행은 평소와 같이 명령 프롬프트에서 `gdb`라고 실행하면 된다. 그럼 자동으로 `gdb-peda`가 load되면서 기능을 사용할 수 ㅇㅆ다.

# 3. gdb-peda 기능 및 명령어
- `gdb-peda` 에서는 아래와 같은 명령어를 사용할 수 있다.
- 또한 아래의 명령어는 `gdb` 를 실행한 후 나오는 `gdb prompt`에서 입력하여 사용할 수 있다.₩
| Command      | Description                                                |
|--------------|------------------------------------------------------------|
| aslr         | GDB의 ASLR 설정을 표시하거나 설정합니다.                  |
| checksec     | 바이너리의 다양한 보안 옵션을 확인합니다.                |
| dumpargs     | 호출 명령어에서 함수에 전달된 인수를 표시합니다.          |
| dumprop      | 지정된 메모리 범위 내의 모든 ROP 가젯을 덤프합니다.     |
| elfheader    | 디버깅 중인 ELF 파일의 헤더 정보를 가져옵니다.         |
| elfsymbol    | ELF 파일에서 디버깅 정보가 아닌 심볼 정보를 가져옵니다. |
| lookup       | 주어진 메모리 범위에 속하는 주소 또는 주소에 대한 참조를 검색합니다. |
| patch        | 주소에서 시작하는 메모리를 문자열/16진수 문자열/정수로 패치합니다. |
| pattern      | 메모리에 주기적인 패턴을 생성, 검색 또는 기록합니다.     |
| procinfo     | /proc/pid/에서 다양한 정보를 표시합니다.                  |
| pshow        | 다양한 PEDA 옵션과 기타 설정을 표시합니다.                |
| pset         | 다양한 PEDA 옵션과 기타 설정을 설정합니다.                |
| readelf      | ELF 파일의 헤더 정보를 가져옵니다.                        |
| ropgadget    | 바이너리 또는 라이브러리의 공통 ROP 가젯을 가져옵니다.   |
| ropsearch    | 메모리에서 ROP 가젯을 검색합니다.                         |
| searchmem    | 메모리에서 패턴을 검색하며 정규식 검색을 지원합니다.    |
| shellcode    | 일반적인 쉘코드를 생성하거나 다운로드합니다.              |
| skeleton     | 파이썬 악용 코드 템플릿을 생성합니다.                    |
| vmmap        | 디버그 중인 프로세스의 섹션에 대한 가상 매핑 주소 범위를 가져옵니다. |
| xormem       | 키로 메모리 영역을 XOR 연산합니다.                         |

# 4. 실행 이미지 (Screenshot)
![start](http://i.imgur.com/P1BF5mp.png)

![pattern arg](http://i.imgur.com/W97OWRC.png)

![patts](http://i.imgur.com/Br24IpC.png)
