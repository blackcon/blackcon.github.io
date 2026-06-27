---
title: Oracle VirtualBox Guest-to-Host VM Escape — VMSVGA DX11 UAF와 LsiLogic 스택 정보 누출 체인
categories: [Research, Exploitation]
tags: [VirtualBox, VM-escape, guest-to-host, UAF, use-after-free, info-leak, VMSVGA, DX11, LsiLogic, ASLR, CFG, DEP, exploit, ZDI, hypervisor]
date: 2026-06-27 21:00:00 +0900
description: VirtualBox 7.2.6의 VMSVGA DX11 백엔드 Use-After-Free(코드 실행)와 LsiLogic SCSI 컨트롤러의 미초기화 스택 정보 누출(ASLR 우회)을 체인하여 게스트에서 호스트로 탈출(Guest-to-Host VM Escape)하는 full exploit을 분석합니다.
pin: false
---

안녕하세요! 이번 글에서는 제가 분석하여 ZDI(Zero Day Initiative)에 제보했던 **Oracle VM VirtualBox**의 게스트→호스트 탈출(Guest-to-Host VM Escape) 취약점 체인을 정리해 보려 합니다.

이 익스플로잇은 두 개의 독립적인 취약점을 엮어서 게스트 OS의 root 권한만으로 호스트의 `VirtualBoxVM.exe` 프로세스에서 임의 코드 실행(`calc.exe` 실행)을 달성합니다.

> **공개 배경**: 이 취약점 체인은 ZDI에 제보했으나, 아래와 같은 회신을 받았습니다.
>
> > _"After reviewing this submission, we have decided not to pursue acquisition of the reported information. This is because we have determined it is a duplicate of an existing vulnerability report we previously acquired."_
>
> 즉 ZDI가 이미 확보한 다른 제보와 **중복(duplicate)**이라는 이유로 acquisition이 거절되었고, 실제로 최신 VirtualBox 소스코드를 확인해 보면 두 취약점 모두 **이미 패치된 상태**입니다. 따라서 더 이상 0-day가 아니며, 학습·연구 목적으로 그 내부 동작을 공유합니다.
{: .prompt-info }

***

# 0. Full Exploit 데모

먼저 전체 체인이 동작하는 영상부터 보시죠. 게스트(Linux, Navix-9.7) 내부에서 익스플로잇을 실행하면, 호스트(Windows 11) 데스크톱에 `calc.exe`가 튀어나옵니다.

<video controls muted playsinline width="100%" style="border-radius:8px;">
  <source src="/assets/img/posts/vbox-vmsvga-uaf/vbox_escape_uaf_full_chain.mp4" type="video/mp4">
  영상을 재생할 수 없습니다. <a href="/assets/img/posts/vbox-vmsvga-uaf/vbox_escape_uaf_full_chain.mp4">여기를 눌러 다운로드</a>하세요.
</video>
_게스트에서 호스트로의 전체 탈출 체인 (LsiLogic leak → VMSVGA UAF → RtlRestoreContext → WinExec("calc.exe"))_

탈출에 성공하면, 호스트의 Process Explorer에서 `calc.exe`가 `VirtualBoxVM.exe`의 **자식 프로세스**로 생성된 것을 확인할 수 있습니다. 이는 게스트가 아니라 호스트 컨텍스트에서 코드가 실행되었다는 결정적인 증거입니다.

![호스트에서 VirtualBoxVM.exe의 자식으로 실행된 calc.exe](/posts/vbox-vmsvga-uaf/exploit-host-calc.png)
_Process Explorer 기준 프로세스 트리: `VBoxSVC.exe → VirtualBoxVM.exe → VirtualBoxVM.exe → VirtualBoxVM.exe → calc.exe`. 게스트에서 트리거한 페이로드가 호스트의 VM 프로세스 컨텍스트(user 권한)에서 `calc.exe`를 spawn했다._

***

# 1. 개요

테스트 대상과 환경은 다음과 같습니다.

| 항목 | 내용 |
|------|------|
| **제품** | Oracle VM VirtualBox |
| **버전** | 7.2.6 r166706 (vulnerable 확인) |
| **호스트** | Windows 11 |
| **게스트** | VMSVGA 3D 가속 + LsiLogic SCSI 컨트롤러를 가진 Linux (root 필요) |
| **결과** | `VirtualBoxVM.exe` 내 임의 코드 실행 (호스트 user 권한) |
| **CVSS 3.1** | 8.8 (High) — AV:L/AC:L/PR:L/UI:N/S:C/C:H/I:H/A:H |

이 글의 핵심인 두 취약점은 다음과 같습니다.

- **Vulnerability A — LsiLogic SCSI 미초기화 스택 정보 누출 (CWE-457)**: 호스트 ASLR을 100% 결정론적으로 우회 (`VBoxVMM.dll` base + 힙 포인터 누출).
- **Vulnerability B — VMSVGA DX11 Context Destroy Use-After-Free (CWE-416)**: 힙 스프레이로 해제된 객체를 재점유하여 COM 인터페이스 포인터를 하이재킹, 임의 코드 실행.

정보 누출(A)로 메모리 레이아웃을 알아낸 뒤, 메모리 손상(B)으로 제어 흐름을 가로채는 전형적인 "leak → corrupt" 구조입니다.

***

# 2. 취약점 분석

## 2.1 Vulnerability A — LsiLogic SCSI 미초기화 스택 누출 (ASLR 우회)

### 근본 원인

문제는 `DevLsiLogicSCSI.cpp`의 `lsilogicR3ProcessSCSIIORequest()` 함수에 있습니다. SCSI I/O 요청이 존재하지 않는 LUN(`u8TargetID >= pThisCC->cDeviceStates`)을 가리킬 때, 함수는 스택 위에 에러 응답을 구성합니다. 이때 60바이트짜리 `MptReplyUnion IOCReply` 변수가 **0으로 초기화되지 않습니다.**

```c
// DevLsiLogicSCSI.cpp:2286 (취약 — SCSI I/O 에러 경로)
MptReplyUnion IOCReply;
// RT_ZERO(IOCReply);   // ← 누락! 28바이트가 게스트로 누출

// DevLsiLogicSCSI.cpp:2238 (정상 — 다른 응답 경로)
MptReplyUnion IOCReply;
RT_ZERO(IOCReply);      // ← 같은 함수 내 다른 경로엔 초기화가 있음
```

동일 함수의 다른 경로(line 2238)에는 `RT_ZERO(IOCReply)`가 있지만, 에러 응답 경로(line 2286)에는 빠져 있는 명백한 개발자 실수입니다.

### 데이터 흐름

```
Guest Input:
  LsiLogic SCSI 도어벨/요청 큐에 I/O port write
  → u8TargetID = 15 (존재하지 않는 디바이스)

Code Flow:
  1. lsilogicIOPortWrite() → lsilogicR3ProcessSCSIIORequest()  [:2271]
  2. MptReplyUnion IOCReply;                  // 60바이트, 미초기화
  3. SCSIIOError 필드 채움 (앞 32바이트만)     // [:2347-2393]
  4. lsilogicFinishAddressReply(...)          // [:2395]
     → min(cbReplyFrame=128, sizeof=60) = 60바이트
     → PDMDevHlpPCIPhysWriteMeta(..., &IOCReply, 60)
     → 게스트가 0x20~0x3B 28바이트(미초기화 호스트 스택)를 읽음
```

앞 32바이트(`SCSIIOError`)만 채워진 채, 나머지 28바이트가 게스트가 접근 가능한 메모리로 DMA를 통해 그대로 복사됩니다.

### 누출되는 데이터

| Offset | Size | 내용 | 설명 |
|--------|------|------|------|
| +0x20 | 8 | `heap_ptr_1` | `pDevIns`/`pThis` 포인터 (힙 주소) |
| +0x28 | 8 | `code_ptr` | `VBoxVMM.dll` 내 리턴 주소 |
| +0x30 | 8 | `heap_ptr_2` | PDM 내부 구조체 포인터 (힙) |
| +0x38 | 4 | `small_const` | 상수값 `0x00000002` |

누출된 `code_ptr`에서 `VBoxVMM.dll` base를 다음과 같이 계산합니다.

```
VBoxVMM_base = code_ptr - 0xAC4F6
```

이 값은 **단일 VM 세션 내에서 안정적**이며, 트리거는 **100% 결정론적**이고 **rate limit이 없어** 무한히 누출 스트림을 뽑을 수 있습니다. 익스플로잇 입장에선 이상적인 정보 누출 프리미티브입니다.

## 2.2 Vulnerability B — VMSVGA DX11 Context Destroy UAF (코드 실행)

### 근본 원인

`DevVGA-SVGA3d-dx-dx11.cpp`의 `vmsvga3dBackDXDestroyContext()`에 있습니다. DX11 컨텍스트가 파괴될 때, 함수는 7종류의 DXVIEW 배열을 순회하며 연결된 COM 객체를 `D3D_RELEASE()`로 해제하고, `RTMemFreeZ()`로 메모리를 해제합니다.

그런데 이 함수는 **각 DXVIEW 노드를 surface의 연결 리스트(`pBackendSurface->listView`)에서 제거(`RTListNodeRemove()`)하는 핵심 단계를 빠뜨립니다.** 그 결과 `listView`에 해제된 메모리를 가리키는 dangling pointer가 남습니다. 이후 surface가 파괴될 때 `vmsvga3dBackSurfaceDestroy()`가 이 stale 리스트를 순회하면서 이미 해제된 DXVIEW 노드를 역참조 → **Use-After-Free**가 발생합니다.

개발자도 이 문제를 인지하고 있었던 흔적이 코드에 남아 있습니다.

```c
/** @todo dxViewDestroy? */   // line 6457
```

올바른 함수 `dxViewDestroy()`(line 1590-1602)는 존재하지만, 컨텍스트 파괴 경로에서는 호출되지 않습니다.

### 데이터 흐름

```
1. vmsvga3dDXDestroyContext() → vmsvga3dBackDXDestroyContext()  [:6382]

2. Context Destroy — DXVIEW Release Loop:
   D3D_RELEASE(paRTV[i].u.pRenderTargetView)   // COM Release만 수행
   RTMemFreeZ(&paRenderTargetView, ...)         // [:6482] 메모리 해제
   // → listView 노드는 제거되지 않음 → DANGLING POINTER

3. [공격자] 힙 스프레이: COTable/MOB 스프레이로 해제된 40바이트 영역을
   게스트 제어 데이터로 재점유, offset +0x10에 가짜 포인터 배치

4. Surface Destroy (이후):
   vmsvga3dBackSurfaceDestroy()                 [:6028]
   RTListForEachSafe(&pBackendSurface->listView, ...)
   → dxViewDestroy(pIter)                        [:6113]
   → D3D_RELEASE(pIter->u.pView)   // UAF: pView가 공격자 제어
   → 가짜 vtable 호출 → CODE EXECUTION
```

### DXVIEW 구조체 (40바이트, x64)

| Offset | Size | 필드 | 설명 |
|--------|------|------|------|
| +0x00 | 4 | `cid` | Context ID |
| +0x04 | 4 | `sid` | Surface ID |
| +0x08 | 4 | `viewId` | View ID |
| +0x0C | 4 | `VMSVGA3DBACKVIEWTYPE` | View type enum |
| **+0x10** | **8** | **`ID3D11View* u.pView`** | **COM 인터페이스 포인터 — 하이재킹 타깃** |
| +0x18 | 8 | `RTLISTNODE.pNext` | 연결 리스트 next |
| +0x20 | 8 | `RTLISTNODE.pPrev` | 연결 리스트 prev |

힙 스프레이 후 offset **+0x10**에 공격자가 만든 가짜 COM vtable을 가리키는 포인터를 배치하면, `D3D_RELEASE()`가 `pView->Release()`를 호출할 때 가짜 vtable을 따라 공격자 코드로 흐릅니다.

영향을 받는 view type은 동일한 unlink 누락 패턴 때문에 7종 모두입니다: RenderTargetView, DepthStencilView, ShaderResourceView, UnorderedAccessView, VideoDecoderOutputView, VideoProcessorInputView, VideoProcessorOutputView.

***

# 3. 익스플로잇 기법

## 3.1 우회한 보호기법

전체 체인은 최신 Windows 완화 기법을 모두 우회합니다.

| 방어 기법 | 상태 | 우회 방법 |
|-----------|------|-----------|
| **ASLR** (17+ bits) | Bypassed | Vuln A — LsiLogic 스택 누출로 `VBoxVMM.dll` base + 힙 포인터 획득 |
| **CFG** (Control Flow Guard) | Bypassed | `TRPMR3Term` (VBoxVMM+0x76d80): `xor eax,eax; ret` — CFG-valid 타깃 |
| **DEP** (Data Execution Prevention) | Bypassed | `RtlRestoreContext` (kernel32 RVA 0x4FFC0) + IRETQ 컨텍스트 피벗 |
| **CET** (Shadow Stack) | N/A | `VirtualBoxVM.exe`에 CET 미적용 — 스택 피벗 가능 |

## 3.2 전체 체인 흐름

```
1. LsiLogic leak       → VBoxVMM base + heap2(pDevIns) 포인터
2. IAT probe           → kernel32.dll base (VBoxVMM IAT 엔트리 경유)
3. EGL context create  → DXVIEW 할당, surface->listView에 연결
4. Context destroy     → DXVIEW free, listView에 dangling pointer
5. COTable/MOB spray   → 해제된 40바이트 영역을 제어 데이터로 재점유
6. Surface destroy     → UAF: dxViewDestroy → 가짜 vtable → TRPMR3Term (CFG 우회)
7. RtlRestoreContext   → IRETQ로 공격자 CONTEXT 구조체로 피벗
8. WinExec("calc.exe") → 호스트에서 임의 코드 실행
```

핵심 아이디어는 다음과 같습니다.

1. **leak으로 ASLR을 깬다.** LsiLogic 누출에서 얻은 `VBoxVMM.dll` base로 모든 가젯 주소를 확정합니다. 추가로 VBoxVMM의 IAT 엔트리를 읽어 `kernel32.dll` base를 알아내고, 여기서 `WinExec`와 `RtlRestoreContext`의 주소를 계산합니다.
2. **재점유 가능한 정확한 크기로 free를 만든다.** DXVIEW는 40바이트이므로, COTable/MOB 스프레이로 같은 크기의 LFH 버킷을 채워 해제 직후 그 슬롯을 게스트 제어 데이터로 덮습니다.
3. **+0x10을 가짜 COM 객체로 채운다.** `pView`가 가짜 vtable을 가리키게 하면, `D3D_RELEASE` → `Release()` 가상 호출이 첫 번째 가젯으로 점프합니다.
4. **CFG를 만족하는 가젯으로 시작한다.** 간접 호출 타깃은 CFG가 검증하므로, 정당한 함수 진입점인 `TRPMR3Term`(`xor eax,eax; ret`)을 경유한 뒤 `RtlRestoreContext`의 IRETQ로 완전한 레지스터/스택 제어를 확보합니다.
5. **CONTEXT를 복원해 페이로드로 점프한다.** `RtlRestoreContext`에 공격자가 구성한 `CONTEXT` 구조체를 넘겨 RIP/RSP/인자 레지스터를 한 번에 세팅하고 `WinExec("calc.exe")`를 호출합니다.

## 3.3 실제 실행 화면

아래는 게스트 내부에서 `run_full_chain.sh`로 전체 체인을 구동하는 장면입니다. 배경의 Process Explorer에는 호스트의 `VirtualBoxVM.exe` 트리가 보입니다.

![게스트 VM 내부에서 실행 중인 full chain 익스플로잇](/posts/vbox-vmsvga-uaf/exploit-guest-running.png)
_게스트(Navix-9.7) 터미널: `MODE 12: AUTO (kernel32 leak → RCE, single session)`. TARGET 컨텍스트 생성(5 views) → FIFO ping으로 채널 확인 → 힙 스프레이(`pView`/`pNext`/`pPrev` 배치, `D3D_RELEASE → RtlCaptureCtx` 유도) → `pView at +0x10` 페이로드 배치(OK) → MOB 스프레이로 CID 255→1 DESTROY를 트리거하는 단계가 로그에 그대로 드러난다._

로그를 보면 익스플로잇의 단계가 그대로 매핑됩니다.

- `Creating TARGET context on fd_A with 5 views` → DXVIEW를 surface에 연결 (3.2의 step 3)
- `[spray] pView → 0x...32d0 (aMsg+0x70 = pThis+0x90)` → +0x10에 들어갈 가짜 객체 포인터 배치 (step 5)
- `[spray] 1st/2nd D3D_RELEASE: RtlCaptureCtx(...)` → 가짜 vtable의 `Release` 슬롯을 가젯으로 유도 (step 6)
- `Payload: pView at +0x10 = 0x...32d0 (OK)` → 재점유 성공, 하이재킹 포인터 안착
- `MOB spray: DESTROY (CIDs 255→1)` → surface destroy를 유발해 UAF 발화 (step 6)

## 3.4 신뢰성

LFH 힙 스프레이의 확률적 특성 때문에 단일 시도 성공률은 **약 67%**입니다. 그래서 재시도 래퍼 스크립트(`run_exploit.sh`, 5회 시도/스프레이 수 조정 가능)를 사용하면 **5회 이내에 사실상 100%**에 도달합니다.

```bash
# 단일 실행 (약 67%)
sudo ./full_chain_exploit

# 재시도 래퍼 (5회 이내 사실상 100%)
sudo ./run_exploit.sh --spray-count 350
```

***

# 4. 패치

두 취약점 모두 수정은 단순합니다. 핵심은 "초기화 누락"과 "unlink 누락"을 메우는 것입니다.

## 4.1 Vulnerability A 패치 — 미초기화 스택 변수 제거

가장 단순한 수정은 선언 직후 `RT_ZERO()`를 추가하는 것입니다. 동일 함수의 다른 경로(line 2238)와 동일한 패턴이죠.

```c
// DevLsiLogicSCSI.cpp:2286
MptReplyUnion IOCReply;
RT_ZERO(IOCReply);        // 1차적 수정: 응답 구조체 전체를 0으로 초기화
```

> **실제 업스트림 패치**: VirtualBox 메인테이너는 `RT_ZERO`를 더하는 대신 **문제의 60바이트 스택 union 자체를 제거**하는 방향으로 더 근본적으로 수정했습니다. `MptReplyUnion IOCReply` 지역 변수를 없애고 `uint16_t u16IOCStatus = 0` 하나만 사용하도록 바꿔, 게스트로 복사되던 미초기화 영역이 코드상 더 이상 존재하지 않게 만들었어요. ([commit `122c6880`](https://github.com/VirtualBox/virtualbox/commit/122c6880d19c312e25a61cb8ba2e46a7a85ff497), 2026-04-10, `bugref:11072`, svn r173576)
{: .prompt-tip }

```c
// 패치 후 (commit 122c6880) — 큰 스택 union 자체가 사라짐
-    MptReplyUnion IOCReply;
-    int rc = VINF_SUCCESS;
+    uint16_t    u16IOCStatus = 0;
+    int         rc = VINF_SUCCESS;
     ...
-                IOCReply.SCSIIOError.u16IOCStatus = MPT_SCSI_IO_ERROR_IOCSTATUS_DEVICE_NOT_THERE;
+                u16IOCStatus = MPT_SCSI_IO_ERROR_IOCSTATUS_DEVICE_NOT_THERE;
```

어느 쪽이든 채워지지 않은 28바이트가 호스트 스택 잔여 데이터로 누출되던 경로가 차단됩니다.

## 4.2 Vulnerability B 패치 — free 이전에 노드 unlink

DXVIEW 배열을 `RTMemFreeZ()`로 해제하기 전에, 각 활성 view에 대해 `dxViewDestroy()`를 호출하여 surface의 `listView`에서 노드를 **먼저 제거**해야 합니다. `dxViewDestroy()`는 unlink와 release를 함께 처리합니다.

```c
// vmsvga3dBackDXDestroyContext() 내, RTMemFreeZ 호출 직전
for (uint32_t i = 0; i < cRTVs; ++i)
{
    if (paRenderTargetView[i].u.pRenderTargetView)
    {
        dxViewDestroy(&paRenderTargetView[i]);  // unlink + release 동시 수행
    }
}
RTMemFreeZ(&paRenderTargetView, sizeof(DXVIEW) * cRTVs);
```

7종 view type 모두 같은 처리를 적용하면, surface 파괴 시 `listView`에 dangling pointer가 남지 않으므로 UAF가 제거됩니다.

> **실제 업스트림 패치**: VirtualBox는 [commit `d691aa68`](https://github.com/VirtualBox/virtualbox/commit/d691aa680c11f51d1ce1f90ee3b198cfc3262f76)(2026-06-09, `bugref:11092`, svn r174129, _"use a helper to delete resource views"_)로 이 문제를 수정했습니다. `vmsvga3dBackDXDestroyContext()`에서 7종 view 전부에 대해 `D3D_RELEASE(...)` 호출을 **`dxViewDestroy(...)`로 교체**했고, 보고서에서 인용했던 바로 그 `/** @todo dxViewDestroy? */` 주석까지 삭제했어요. 즉 제가 제안했던 수정 방향과 **정확히 일치**합니다.
{: .prompt-tip }

```c
// 패치 후 (commit d691aa68) — 모든 view를 dxViewDestroy로 해제 (unlink 포함)
     for (uint32_t i = 0; i < pBackendDXContext->cRenderTargetView; ++i)
-        D3D_RELEASE(pBackendDXContext->paRenderTargetView[i].u.pRenderTargetView);
+        dxViewDestroy(&pBackendDXContext->paRenderTargetView[i]);
     ...
-    /** @todo dxViewDestroy? */
     for (uint32_t i = 0; i < pBackendDXContext->cVideoDecoderOutputView; ++i)
-        D3D_RELEASE(pBackendDXContext->paVideoDecoderOutputView[i].u.pVideoDecoderOutputView);
+        dxViewDestroy(&pBackendDXContext->paVideoDecoderOutputView[i]);
```

## 4.3 패치 상태

서두에 적었듯 ZDI는 이 제보를 기존 확보 건과 **중복**으로 판단했고, 실제로 [공식 VirtualBox 소스 저장소](https://github.com/VirtualBox/virtualbox)에서 두 취약점 모두 수정된 것을 확인할 수 있습니다.

| 취약점 | 패치 커밋 | 날짜 | bugref / svn |
|--------|-----------|------|--------------|
| Vuln A (LsiLogic leak) | [`122c6880`](https://github.com/VirtualBox/virtualbox/commit/122c6880d19c312e25a61cb8ba2e46a7a85ff497) | 2026-04-10 | `bugref:11072` / r173576 |
| Vuln B (VMSVGA DX11 UAF) | [`d691aa68`](https://github.com/VirtualBox/virtualbox/commit/d691aa680c11f51d1ce1f90ee3b198cfc3262f76) | 2026-06-09 | `bugref:11092` / r174129 |

따라서 현재 최신 버전을 사용 중이라면 영향을 받지 않습니다. VirtualBox는 항상 최신 버전으로 유지하시길 권장드립니다.

***

# 5. 마치며

이번 체인은 "leak → corrupt"라는 익스플로잇의 정석을 하이퍼바이저 경계에서 그대로 보여주는 사례였습니다.

- **장치 에뮬레이션(LsiLogic)의 사소한 초기화 누락**이 결정론적 ASLR 우회로 이어지고,
- **3D 백엔드(VMSVGA DX11)의 연결 리스트 unlink 누락**이 신뢰할 수 있는 UAF → RCE로 이어졌습니다.

특히 두 취약점 모두 코드상에 "정답이 바로 옆 경로에 존재"(`RT_ZERO`, `dxViewDestroy`)했다는 점이 인상적입니다. 복잡한 로직 버그가 아니라 일관성 누락 한 줄이 게스트→호스트 탈출의 결정적 디딤돌이 된 것이죠.

가상화 장치 에뮬레이션 코드를 감사하실 때, **게스트로 되돌아가는 모든 버퍼의 초기화**와 **객체 수명/연결 리스트 정리**를 우선 점검 대상으로 두면 비슷한 클래스의 버그를 효율적으로 찾을 수 있을 거라 생각합니다.

읽어주셔서 감사합니다! 🙏

***

# 6. 참고 링크 (References)

**공식 소스 저장소**
- VirtualBox 소스 (GitHub 미러): <https://github.com/VirtualBox/virtualbox>

**취약 코드 (main 기준 현재 상태 — 이미 패치됨)**
- LsiLogic SCSI: [`src/VBox/Devices/Storage/DevLsiLogicSCSI.cpp`](https://github.com/VirtualBox/virtualbox/blob/main/src/VBox/Devices/Storage/DevLsiLogicSCSI.cpp)
- VMSVGA DX11 백엔드: [`src/VBox/Devices/Graphics/DevVGA-SVGA3d-dx-dx11.cpp`](https://github.com/VirtualBox/virtualbox/blob/main/src/VBox/Devices/Graphics/DevVGA-SVGA3d-dx-dx11.cpp)

**패치 커밋**
- Vuln A — LsiLogic 미초기화 스택 제거: [`122c6880`](https://github.com/VirtualBox/virtualbox/commit/122c6880d19c312e25a61cb8ba2e46a7a85ff497) (`bugref:11072`, svn r173576)
- Vuln B — VMSVGA DX11 view 해제 헬퍼 사용(UAF fix): [`d691aa68`](https://github.com/VirtualBox/virtualbox/commit/d691aa680c11f51d1ce1f90ee3b198cfc3262f76) (`bugref:11092`, svn r174129)

**제품 다운로드**
- VirtualBox 다운로드(최신): <https://www.virtualbox.org/wiki/Downloads>
- 분석 대상 7.2.6 (구버전): <https://www.virtualbox.org/wiki/Download_Old_Builds_7_2>
