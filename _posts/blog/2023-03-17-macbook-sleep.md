---
title: 맥북 덮개를 덮어도 꺼지지 않게 설정하는 방법
categories: [macos, tips]
tags: [macos, macbook, tips]
date: 2023-03-16 23:05:00 +0900
---

재택 근무가 길어짐에 따라 책상를 정비해야할 일이 있었습니다. 
현재 사용중인 맥북을 모니터에 연결하여 setup을 하고 싶었고, 
여느 노트북과 같이 맥북에 모니터 케이블을 연결한 후 덮개를 닫으면 깔끔하게 사용할 수 있을 줄 알았습니다. 
하지만 덮개를 닫으면 절전모드로 진입하는 맥북... 
맥북이 절전되지 않도록 설정하는 방법을 공유드립니다.

# Summary
1. Terminal (혹은 iterm)을 연다.
2. Terminal 상에서 `sudo pmset -c disablesleep 1` 명령어를 입력한다. (맥북이 잠들지 않게 함)
3. 사용자의 비밀번호를 요구하면 입력 후에 `Enter`를 치면 끝이다.
4. 다시 맥북을 잠들게 하려면 `sudo pmset -c disablesleep 0` 명령어를 입력한다.

# Detail
맥북을 개발용도로 사용하지 않는 user도 있으시기에 `terminal` 실행부터 차근차근 작성해보겠습니다.
1. Terminal 열기
   - spotligt 검색 열기: Command + Spacebar<br>
      ![mac-sleep-1](/posts/mac-sleep-1.png)
   - 검색 창에 `터미널` 혹은 `iterm` 입력 후 엔터<br>
      ![mac-sleep-2](/posts/mac-sleep-2.png)
2. 명령어 입력하기 (맥북이 잠들지 않게 함)
    ```bash
    ➜  ~ sudo pmset -c disablesleep 1
    Password:
    ```
3. 맥북 덮어보기 (테스트!)
# Share
저는 위 명령어를 평소에 사용하지 않다보니 외워지지도 않습니다. 
그래서 저는 간단한 `shell script`를 작성하여 컴퓨터에 저장을 해두었구요.
상황에 따라서 `on` 또는 `off` 옵션을 함께 입력하여 기능을 제어할 수 있도록 했습니다.
- shell script
  ```bash
  #!/bin/bash

  if [ $# -ne 1  ]; then
      echo "Usage: $0 on|off"
      exit -1
  else
      if [ $1 == "on" ]; then
          sudo pmset -c disablesleep 1;
      elif [ $1 == "off"  ];then
          sudo pmset -c disablesleep 0;
      else
          echo "Invalid option"
      fi
  fi
  ```
- 사용법
  - 덮개를 덮어도 잠들지 않게 하기
   ```bash
   ./sleep_mac.sh on
   ```
  - 덮개를 덮어도 잠들게 하기
   ```bash
   ./sleep_mac.sh off
   ```
