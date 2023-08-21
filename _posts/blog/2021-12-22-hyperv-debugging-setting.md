---
title: Hyper-V 분석하기 위한 환경설정 하는 방법
categories: [Hacking, Reversing]
tags: [Hacking, Reversing]
date: 2021-12-22 18:05:00 +0900
---
## 들어가기에 앞서..

해당 포스트는 MSRC(Microsoft Security Response Center) 블로그에 작성된 [First Step Hyper-V Research](https://msrc-blog.microsoft.com/2018/12/10/first-steps-in-hyper-v-research/) 내용을 토대로 작성하였습니다.

## Debugging Environment

### 1) Intro

이 글에서 작성할 환경 설정은 nested(중첩) VM을 생성하고 이 내부에서 Hyper-V guest의 하이퍼바이저와 루트 파티션의 커널을 디버깅하기 위함입니다. **Hyper-V**는 [하이퍼바이저 중에서도 Type-1 방식](http://cloudrain21.com/hypervisor-types)이기 때문에 Host 에서는 커널과 하이퍼바이저를 디버깅할 수 없습니다. 이를 위해 게스트를 만들고 그 안에 Hyper-V를 활성화([nested vm](https://docs.microsoft.com/ko-kr/virtualization/hyper-v-on-windows/user-guide/nested-virtualization))하고 모든 것을 구성하여 디버깅을 할 것입니다. 다행히도 Hyper-V는 이 방식에서 활용할 중첩 가상화를 지원합니다. 디버깅 환경은 아래 이미지와 같습니다.

![debugging_env](/posts/hyperv-debug-2018-12-arch.png)

디버깅하려는 하이퍼바이저 내부에 다른 하이퍼바이저의 게스트로 실행합니다. 간단히 말해서 L0 루트 파티션의 사용자 공간에서 L1 하이퍼바이저와 루트 파티션의 커널을 디버그합니다.

**<Nested VM 용어 설명>**

-   L0 = 물리적 호스트에서 실행되는 코드. 하이퍼바이저를 실행합니다.
-   L1 = L0의 하이퍼바이저 게스트. 디버그하려는 하이퍼바이저를 실행합니다.
-   L2 = L1의 하이퍼바이저 게스트.

Hyper-V 의 중첩 가상화에 대하여 좀 더 상세한 설명은 [여기](https://docs.microsoft.com/ko-kr/virtualization/hyper-v-on-windows/user-guide/nested-virtualization)를 참조해주세요 :)

### 2) Setting for debugging

하이퍼바이저에 디버그 지원이 내장되어 있어 디버거로 Hyper-V에 연결할 수 있습니다. 활성화하려면 BCD(Boot Configuration Data) 변수에서 몇 가지 설정을 구성해야 합니다.

**<디버깅 할 VM 설정하기>**

1.  **Hyper-V 가 활성화되어 있지 않다면 활성화하기(Level0)**
    1.  [이 문서](https://docs.microsoft.com/ko-kr/virtualization/hyper-v-on-windows/quick-start/enable-hyper-v)를 참고하여 Hyper-v 활성화
    2.  Host OS 재부팅하기
2.  **디버깅을 위해 새로운 guest VM을 설정하기(Level1)**
    1.  [여기에](https://docs.microsoft.com/ko-kr/virtualization/hyper-v-on-windows/quick-start/create-virtual-machine) 설명된 대로 1세대 Windows 10 게스트를 만듭니다. (2세대 guest에서도 작동하지만 'Secure boot' 를 비활성화해야 함)
        -   참고 이미지 (nested VM)  
            ![nested-vm](/posts/hyperv-debug-nested-vm.png)
    2.  게스트 프로세서에서 VT-x를 활성화합니다. 활성화하지 않으면 Hyper-V 플랫폼이 게스트 내부에서 실행되지 않습니다.  
        (L1 _Guest OS의 전원을 끄고 L0 Host에서 PowerShell(관리자 권한) 에서 활성화 가능_)
        -   command  
            ```
            Set-VMProcessor -VMName <VMName> -ExposeVirtualizationExtensions $true​
            ```
            
        -   참고 이미지  
            ![hyperv-set-1](/posts/hyperv-debug-setting.png)
    3.  L1게스트 내부에서 하이퍼바이저를 디버깅할 것이므로 L1게스트 내부에서도 Hyper-V를 활성화해야 합니다( [여기에](https://docs.microsoft.com/en-us/virtualization/hyper-v-on-windows/quick-start/enable-hyper-v) 문서화됨 ). 나중에 게스트 VM을 재부팅합니다.
    4.  이제 디버깅을 활성화하기 위해 [BCD 변수](https://docs.microsoft.com/ko-kr/windows-server/administration/windows-commands/bcdedit)를 설정해야 합니다. L1 게스트(방금 설정한 내부 OS) 내부에서 cmd.exe (관리자 권한) 에서 다음을 실행합니다. 아래와 같이 설정을 하면 부팅 후에 적용됩니다
        -   command  
            ```
            # 직렬 포트를 통해 Hyper-V 디버깅 활성화
            bcdedit /hypervisorsettings serial DEBUGPORT:1 BAUDRATE:115200
            bcdedit /set hypervisordebug on
            
            # Hyper-V가 부팅 시 실행되도록 활성화하고 커널을 로드
            bcdedit /set hypervisorlaunchtype auto
            
            # 다른 직렬 포트를 통해 커널 디버그 활성화
            bcdedit /dbgsettings serial DEBUGPORT:2 BAUDRATE:115200
            bcdedit /debug on
            
            # 테스트 커널 드라이버를 load 할 일이 있을 경우, 아래 기능 활성화
            bcdedit /set TESTSIGN on
            ```
            
        -   참고 이미지  
            ![hyperv-set-2](/posts/hyperv-debug-setting-2.png)
3.  **앞서 설정한 직렬포트와 연결하기 위해서는 Level 0 VM에서 아래와 같이 셋팅한다.**
    1.  Level 1 VM (guestOS) 종료하기
    2.  Level 1 VM 설정하기
        -   Hyper-V 관리자에서 Level1 VM 오른쪽 버튼 클릭 -> \[설정\] -> 왼쪽 탭에서 \[하드웨어\] -> \[COM1\]을 선택
        -   Named Pipe를 선택하고 "debug\_kernel"로 설정
        -   COM2에 대해서도 동일한 작업을 수행하고 이름을 "debug\_hv"로 설정합니다.
        -   _참고:  [Set-VMComPort](https://docs.microsoft.com/en-us/powershell/module/hyper-v/set-vmcomport?view=win10-ps) cmdlet를 사용해서도 위와 같은 설정을 할 수 있음_
    3.  이제 Named Pipe 로써 “\\\\.\\pipe\\debug\_hv” 와 “\\\\.\\pipe\\debug\_kernel” 이 생성되었고, 각각 hypervisor와 루트 파티션(Level1 VM) 커널에 디버깅을 할 수 있게 되었다.
        -   참고 이미지  
            ![hyperv-set-3](/posts/hyperv-debug-setting-3.png)
    4.  **Level 0 VM에서 2개의 windbg를 open하여 named pipe로 연결한다.**
        -   windbg 열기 -> 디버깅 시작-커널 연결 (ctrl+k)  
            ![hyperv-set-4](/posts/hyperv-debug-setting-4.png)
    5.  **guest OS (level1 VM)을 재부팅하면 디버깅이 가능하다.**
        -   참고 이미지  
            ![hyperv-set-5](/posts/hyperv-debug-setting-5.png)

### 3) EOD (End-Of-Documentation)

여기까지 Hyper-V의 커널과 하이퍼바이저를 디버깅하기 위해서 어떻게 해야하는지 작성해보았는데요. 이 내용은 MS 내부 직원이 분석하는 방법중 하나로써 MS블로그 글에 기재가 되어 있었습니다.

블로그 글에서 이 방법뿐만 아니라 다른 방법에 대한 링크를 아래와 같이 공유가 되었으니 참고해주세요 :)

-   [VMware 와 IDA를 이용하는 방법](http://hvinternals.blogspot.com/2015/10/hyper-v-debugging-for-beginners.html) ( [@gerhart\_x](https://twitter.com/gerhart_x) )
    
    [Hyper-V debugging for beginners
    
    hyper-v internals
    
    hvinternals.blogspot.com](http://hvinternals.blogspot.com/2015/10/hyper-v-debugging-for-beginners.html)
    
-   [KDNET::네트워킹을 이용한 디버깅 방법](https://docs.microsoft.com/ko-kr/windows-hardware/drivers/debugger/setting-up-a-network-debugging-connection)   
    
    [수동으로 KDNET 네트워크 커널 디버깅 설정 - Windows drivers
    
    디버깅 도구 for Windows 네트워크를 통해 커널 디버깅을 지원합니다.
    
    docs.microsoft.com](https://docs.microsoft.com/ko-kr/windows-hardware/drivers/debugger/setting-up-a-network-debugging-connection)