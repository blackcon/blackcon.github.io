---
title: "CVE-2026-64777 — apple/container BuildFSSync 경로 컨테인먼트 우회로 호스트 파일 유출 (Claude Code SAST 세션 회고)"
categories: [Research, Exploitation]
tags: [apple-container, CVE-2026-64777, GHSA, path-traversal, symlink, guest-to-host, BuildKit, Swift, SAST, Claude-Code, AI-agent, bugbounty, duplicate]
date: 2026-08-16 21:00:00 +0900
description: Apple의 macOS 컨테이너 런타임 apple/container에서 빌드 컨텍스트 밖의 호스트 파일이 유출되는 CVE-2026-64777(GHSA-2v2q-4q35-h585)을 분석하고, 이 결함을 Claude Code 기반 SAST 파이프라인이 어떻게 찾아냈는지 세션 로그를 그대로 복기합니다. 중복 제보였지만 크레딧에는 이름이 올라갔습니다.
pin: false
---

안녕하세요! 이번 글에서는 제가 분석하여 Apple에 제보했던 macOS용 컨테이너 런타임 [**apple/container**](https://github.com/apple/container)의 **빌드 컨텍스트 이탈 → 호스트 파일 유출** 취약점을 정리해 보려 합니다.

- **CVE**: CVE-2026-64777
- **Advisory**: [GHSA-2v2q-4q35-h585](https://github.com/apple/container/security/advisories/GHSA-2v2q-4q35-h585)
- **제목**: *Build filesystem sync discloses host files outside the build context via symlinks*
- **영향 버전**: `<= 1.1.0`
- **패치 버전**: `1.2.0`
- **심각도**: Moderate

> **공개 배경**: 저도 이 이슈를 제보했지만, **이미 접수된 다른 제보와 중복(duplicate)** 이었습니다. 다만 Apple 측에서 중복 제보자들도 함께 인정해 주어 어드바이저리의 **Credits 섹션에 이름을 함께 올려 주었습니다**. 이미 `1.2.0`에서 패치가 배포된 사안이라 더 이상 0-day가 아니며, 학습·연구 목적으로 내부 동작과 발견 과정을 공유합니다.
{: .prompt-info }

글은 크게 두 축으로 나눠서 풀어보려 합니다.

1. **취약점 자체** — 왜 호스트가 게스트에게 파일을 서빙하는 구조가 되었고, 어디서 컨테인먼트가 깨졌는가
2. **발견 과정** — Claude Code 기반 SAST 멀티에이전트 파이프라인이 이 파일을 어떻게 후보로 올리고, 어떻게 결함을 특정했는가 (실제 세션 로그 복기)

***

# 1. apple/container의 빌드 아키텍처

apple/container는 Docker Desktop과 달리 **컨테이너 하나당 경량 VM 하나**를 띄우는 구조입니다. 이미지 빌드도 마찬가지로, 실제 빌드 엔진(BuildKit)은 **게스트 VM 안**에서 돌아갑니다.

여기서 재미있는 역전이 일어납니다. 빌드 컨텍스트(`docker build .`의 그 `.`)는 **호스트**에 있는데, 이걸 소비하는 BuildKit은 **게스트** 안에 있습니다. 그래서 호스트 쪽 프로세스가 **파일 서버 역할**을 하고, 게스트의 BuildKit이 gRPC 스트림으로 "이 파일 줘"라고 요청하는 구조가 됩니다.

```text
   [ 호스트 macOS ]                          [ 게스트 VM ]
                                                    
   빌드 컨텍스트 (./)                          BuildKit (빌드 엔진)
        │                                            │
   BuildFSSync (파일 서버)  ◄──── gRPC ServerStream ──┘
        │                        BuildTransfer { source, offset, len, ... }
        ├─ walk()   : 컨텍스트 트리 나열
        ├─ info()   : 파일 메타데이터 반환
        └─ read()   : 파일 내용 반환
```

즉 **신뢰 경계의 방향이 뒤집혀 있습니다.** 보통 우리는 "게스트가 호스트로부터 격리되어 있다"고 생각하지만, 여기서는 **게스트가 호스트에게 파일 요청을 보내는 클라이언트**이고, 호스트가 그 요청을 처리하는 서버입니다. 그리고 그 요청 파라미터인 `BuildTransfer.source`는 **게스트가 완전히 제어**합니다.

관련 protobuf 필드는 다음과 같습니다.

```swift
// Builder.pb.swift
struct BuildTransfer {
    var source: String   // "The absolute path to the source from the server perspective."
    // ...
}
```

주석에 "absolute path"라고 적혀 있는 것부터가 이미 불길하죠. 실제로 `read()`/`info()` 구현은 절대 경로를 그대로 받아들입니다.

***

# 2. 취약점 — `walk()`는 하는데 `read()`/`info()`는 안 하는 것

## 2.1 취약한 코드 (v1.1.0 이하)

`Sources/ContainerBuild/BuildFSSync.swift`의 세 핸들러를 나란히 놓고 보면 결함이 바로 드러납니다.

```swift
func read(_ sender: AsyncStream<ClientStream>.Continuation, _ packet: BuildTransfer, _ buildID: String) async throws {
    let offset: UInt64 = packet.offset() ?? 0
    let size: Int = packet.len() ?? 0
    var path: URL
    if packet.source.hasPrefix("/") {
        path = URL(fileURLWithPath: packet.source).standardizedFileURL     // ← 절대 경로 그대로
    } else {
        path = contextDir.appendingPathComponent(packet.source).standardizedFileURL
    }
    if !FileManager.default.fileExists(atPath: path.cleanPath) {
        path = URL(filePath: self.contextDir.cleanPath)
        path.append(components: packet.source.cleanPathComponent)          // ← percent-decoding 후 재조립
    }

    // ★ 컨테인먼트 검사 없이 곧바로 디스크 I/O
    let data = try {
        if try path.isDir() { return Data() }
        let file = try LocalContent(path: path.standardizedFileURL)
        return try file.data(offset: offset, length: size) ?? Data()
    }()

    // 검사는 여기서, 그것도 "이름" 기준으로만
    let transfer = try path.buildTransfer(id: packet.id, contextDir: self.contextDir, complete: true, data: data)
    // ...
}
```

`info()`도 동일한 패턴이고, 반면 `walk()`는 제대로 되어 있습니다.

```swift
func walk(...) async throws {
    // ...
    for url in followPathsWalked {
        guard self.contextDir.parentOf(url) else { continue }              // ← 컨테인먼트 검사
        // ...
        if url.isSymlink {
            let target: URL = url.resolvingSymlinksInPath()                // ← 심볼릭 링크 해석
            if self.contextDir.parentOf(target) {                          // ← 해석된 타겟 재검사
                // 컨텍스트 안일 때만 엔트리에 포함
            }
        }
    }
}
```

**`walk()`는 심볼릭 링크를 해석해서 다시 검사하는데, `read()`와 `info()`는 그렇지 않습니다.** 이 비대칭이 취약점의 전부입니다.

## 2.2 `parentOf()`는 "문자열" 검사다

핵심은 컨테인먼트 헬퍼가 실제 파일이 아니라 **경로 문자열**만 본다는 점입니다.

```swift
// Sources/ContainerBuild/URL+Extensions.swift
fileprivate var fs_components: [String] {
    var parts: [String] = []
    for segment in self.split(separator: "/", omittingEmptySubsequences: true) {
        switch segment {
        case ".":  continue
        case "..": if !parts.isEmpty { parts.removeLast() }   // ← 문자열 레벨에서만 .. 처리
        default:   parts.append(String(segment))
        }
    }
    return parts
}

func parentOf(_ url: URL) -> Bool {
    let parentParts = self.absoluteURL.cleanPath.fs_components
    let childParts  = url.absoluteURL.cleanPath.fs_components
    guard parentParts.count <= childParts.count else { return false }
    return zip(parentParts, childParts).allSatisfy { $0 == $1 }   // ← prefix 비교
}
```

`fs_components`는 `..`를 **텍스트로** 접습니다. `realpath(3)`도, `resolvingSymlinksInPath()`도 호출하지 않습니다. 즉 이 함수가 보장하는 것은 오직 하나입니다.

> "요청된 **이름**이 컨텍스트 디렉터리 아래에 있는 것처럼 **보인다**"

파일이 **실제로 어디에 있는지**는 전혀 검사하지 않습니다. 어드바이저리가 짚은 근본 원인도 정확히 이 문장입니다.

> *"the host-side actor that serves these requests enforces containment on the lexical path of the requested name — it checks whether the name is textually inside the context directory, not whether the file it ultimately resolves to is."*

## 2.3 익스플로잇 시나리오

두 가지 요청 형태를 나눠서 보면 결과가 다릅니다.

**(A) 절대 경로 요청 — 호스트에서 읽히지만 반출은 막힘**

```text
게스트 BuildKit → BuildTransfer { source: "/etc/passwd", method: read }

호스트 read():
  1. path = URL(fileURLWithPath: "/etc/passwd")        ← 검사 없음
  2. LocalContent(path:).data()                        ← 호스트 디스크에서 실제로 읽힘 (I/O 발생)
  3. buildTransfer() → relativeChildPath(to: contextDir)
       → parentOf() 실패 → throw                       ← 데이터는 게스트로 안 나감
```

여기까지는 "검사 순서가 잘못됐다(TOCTOU성 read-before-check)"에 그칩니다. 파일 존재 여부나 오류/타이밍 기반 오라클은 되지만 내용 반출은 안 됩니다.

**(B) 심볼릭 링크 요청 — lexical 검사를 그대로 통과**

여기서부터가 실제 정보 유출입니다. 빌드 컨텍스트 안에 컨텍스트 밖을 가리키는 심링크가 있으면 됩니다.

```bash
# 빌드 컨텍스트 준비 (악성 저장소를 clone 받아 빌드하는 상황을 가정)
$ ls -la ./ctx
lrwxr-xr-x  1 user  staff   11  keys -> /Users/user/.ssh
lrwxr-xr-x  1 user  staff   11  cfg  -> /Users/user/.aws/credentials
```

```text
게스트 BuildKit → BuildTransfer { source: "cfg", method: read }

호스트 read():
  1. path = contextDir + "cfg"                          → /.../ctx/cfg
  2. standardizedFileURL 은 심링크를 해석하지 않음        ← 이름은 그대로 ctx 아래
  3. LocalContent(path:).data()                         ← 커널이 심링크를 따라가 실제 파일을 읽음
  4. parentOf("/.../ctx/cfg") == true                   ← 이름 기준이라 통과!
  5. 게스트로 ~/.aws/credentials 내용 전송               ← 유출 성립
```

`info()`도 같은 방식으로 컨텍스트 밖 파일의 mode/size/uid/gid/mtime을 흘립니다. 실제 데이터 유출에 앞서 **정찰 프리미티브**로 쓰기 딱 좋은 형태죠.

**공격자 위치**는 어드바이저리 표현 그대로 "malicious builder peer"입니다. 실무적으로는 다음 두 가지가 현실적인 진입점이라고 생각합니다.

- 신뢰할 수 없는 저장소를 clone 받아 `container build` 하는 경우 (심링크는 git이 그대로 보존합니다)
- 게스트 VM 내 BuildKit이 손상되어 임의의 `BuildTransfer` 패킷을 만들 수 있는 경우

어느 쪽이든 **빌드 컨텍스트 경계를 넘어 호스트 사용자 권한으로 읽을 수 있는 모든 파일**이 대상이 됩니다. 브라우저 쿠키, SSH 키, 클라우드 크리덴셜 모두 사정권입니다.

***

# 3. 발견 과정 — Claude Code SAST 파이프라인 세션 복기

이 결함은 제가 코드를 직접 읽다가 찾은 게 아니라, 평소 만들어 쓰던 **Claude Code 기반 SAST 멀티에이전트 파이프라인**이 올려준 finding에서 출발했어요. 다행히 세션 로그가 그대로 남아 있어서, 어떤 경로로 이 파일까지 도달했는지 복기해 봤습니다.

| 항목 | 값 |
|------|-----|
| 실행 일자 | 2026-06-11 |
| 모델 | `claude-opus-4-6` |
| 타깃 | apple/container @ `6508ace` (2026-06-09) |
| 규모 | Swift 395 files / 약 40K LOC |

## 3.1 정찰 단계 — 파일이 큐에 3번으로 올라오다

`attack-surface` 단계가 22개 서브트리를 훑고 143개 파일의 감사 우선순위 큐를 만들었는데, 문제의 파일이 **3순위**로 올라왔습니다.

```text
--- Merged queue: 143 unique files ---
    1 | Sources/APIServer/APIServer+Start.swift | Main daemon entry; 30+ XPC route registrations | from:APIServer
    2 | Sources/CAuditToken/include/AuditToken.h | Sole FFI declaration; private API usage       | from:CAuditToken
    3 | Sources/ContainerBuild/BuildFSSync.swift | Path traversal via server-supplied paths at :66-96 | from:ContainerBuild
    4 | Sources/ContainerCommands/BuildCommand.swift | 515 LOC, stdin Dockerfile, secret handling | from:ContainerCommands
    5 | Sources/ContainerLog/OSLogHandler.swift  | `privacy: .public` on all log messages at :76  | from:ContainerLog
```

정찰 에이전트가 이미 `:66-96` 라인 범위까지 특정해서 넘겼습니다. 이 시점에는 아직 "server-supplied path"라는 관찰뿐입니다.

## 3.2 감사 단계 — 3-stage 감사에서 비대칭을 잡아내다

`code-audit` 단계는 5개 파일씩 묶어 서브에이전트에 던지고, 각 에이전트가 **Stage 1(위협 클래스 스캔) → Stage 2(도달 가능성/상위 게이트 확인) → Stage 3(finding 작성)** 순으로 진행합니다. Batch 1 에이전트의 Stage 1 메모입니다.

```text
[File 3: BuildFSSync.swift]
- Path traversal (5): read() and info() methods accept packet.source which can be
  absolute ("/"-prefixed). Line 70: `if packet.source.hasPrefix("/")` allows absolute
  path reads. The `info()` method (line 100) also accepts absolute paths with no
  containment check.
Verdict: PATH TRAVERSAL is a strong candidate. The `walk()` method has
  `contextDir.parentOf(url)` guard (line 145) but `read()` and `info()` for
  absolute paths have NO containment check.
```

Stage 2에서는 상위 호출자를 타고 올라가 **완화 게이트가 있는지** 확인합니다. 여기서 다른 후보 두 개는 떨어졌고 이것만 살아남았습니다.

```text
**Stage 2 verdicts:**
- BuildFSSync path traversal in `read()` and `info()`: SURVIVES.
  No upstream gate. No containment check for absolute paths.
  VM-to-host trust boundary crossing. HIGH confidence.
- APIServer XPC auth: DROPPED. EUID gate exists at XPCServer.handleMessage().
- OSLogHandler privacy: DROPPED. Speculative.
```

같은 에이전트가 `walk()`와의 대조를 명시적으로 기록해 둔 부분이 이 finding의 핵심입니다.

```text
**Contrast:** The `walk()` method properly checks `contextDir.parentOf(url)`
but `read()` and `info()` do not when given absolute paths.
```

## 3.3 최종 finding — 에이전트가 남긴 기록

최종적으로 작성된 finding JSON은 이렇게 생겼습니다 (일부 발췌).

```json
{
  "id": "auditor-pathtraversal-read-abspath",
  "status": "confirmed",
  "category": "path-traversal",
  "severity": "high",
  "confidence": "high",
  "title": "BuildFSSync.read() allows arbitrary host file read via absolute path in packet.source",
  "file": "Sources/ContainerBuild/BuildFSSync.swift",
  "start_line": 66,
  "end_line": 96,
  "trigger_path": "BuildKit VM -> gRPC ServerStream -> BuildPipeline.run() -> BuildFSSync.handle() -> read() with packet.source = \"/etc/passwd\"",
  "threat_theme": "vm-escape-file-read",
  "invariants_violated": [
    "path-containment: read() trusts packet.source absolute paths without verifying containment within contextDir, unlike walk() which checks contextDir.parentOf(url)"
  ],
  "upstream_validation": { "gate_present": false },
  "verdict": "TP",
  "dedup_group": "PATH-TRAV"
}
```

`verify` 스테이지에서 같은 파일의 형제 finding 두 개는 FP로 정리되었습니다.

| Finding ID | Severity | Verdict | 사유 |
|-----------|----------|---------|------|
| `auditor-pathtraversal-read-abspath` | high | **TP** | 컨테인먼트 검사 전에 디스크 I/O 발생 |
| `auditor-pathtraversal-info-abspath` | medium | FP | 하위 `relativeChildPath()`가 절대 경로를 걸러냄 |
| `auditor-pathtraversal-read-pctdecode` | medium | FP | fallback 경로의 percent-decoding, 플랫폼 의존적 |

## 3.4 리포트 단계에서의 결론

`report` 스테이지가 작성한 bug bounty 리포트의 Finding 8은 다음과 같이 정리되었습니다.

```markdown
## Finding 8: BuildFSSync.read() File Read Before Containment Check

**Severity**: Medium
**File**: `Sources/ContainerBuild/BuildFSSync.swift:66-96`
**CWE**: CWE-22

File I/O occurs at line 86-87 **before** the containment check in `buildTransfer()`
at line 90. The containment check does prevent data from being sent back to the
guest (it throws), but the file is already read from disk.
```

그리고 아키텍처 평가 섹션에는 이런 문장이 남아 있습니다.

```markdown
### What's done well
4. **walk() containment**: The `walk()` method correctly applies `parentOf()`
   guards for both regular files and symlink targets.
```

**개인적으로는 여기가 결정적인 지점이었어요.** 파이프라인은 "`walk()`는 심링크 타겟까지 재검사한다"를 *장점*으로 적어 두었는데, 뒤집어 읽으면 결국 `read()`/`info()`에는 그 재검사가 **없다**는 이야기이기도 하니까요. 리포트를 읽다가 이 대칭을 뒤집어 보면서 심링크 시나리오까지 확장했고, 그게 실제 제보로 이어졌습니다.

정리하면 이렇습니다.

- **에이전트가 스스로 도달한 지점**: `read()`/`info()`의 컨테인먼트 검사가 (a) I/O 이후에, (b) 실제 파일이 아닌 경로 문자열에 대해서만 수행된다
- **사람이 보탠 마지막 한 걸음**: 그 lexical 검사를 *통과하면서도* 컨텍스트 밖을 가리키는 입력 = **심볼릭 링크**

AI 파이프라인이 근본 원인(resolved path가 아닌 lexical path 기준의 검증)까지는 스스로 도달했지만, 그걸 "완전한 데이터 반출 프리미티브"로 조립하는 마지막 한 걸음은 리포트를 읽던 사람의 몫이었던 셈이죠. 저는 이 정도의 분업이면 꽤 이상적인 그림이라고 생각합니다.

***

# 4. 패치 (1.2.0)

수정은 단순하고 정확합니다. **I/O 이전에, resolved 경로를 기준으로** 검사하도록 두 함수 모두에 가드를 넣었습니다.

```swift
// read()
    if !FileManager.default.fileExists(atPath: path.cleanPath) {
        path = URL(filePath: self.contextDir.cleanPath)
        path.append(components: packet.source.cleanPathComponent)
    }
+   let resolved = path.resolvingSymlinksInPath()
+   guard self.contextDir.parentOf(resolved) else {
+       throw Error.pathIsNotChild(resolved.cleanPath, self.contextDir.cleanPath)
+   }
    let data = try { /* ... 파일 읽기 ... */ }()

// info()
    let path: URL = /* ... */
+   let resolved = path.resolvingSymlinksInPath()
+   guard self.contextDir.parentOf(resolved) else {
+       throw Error.pathIsNotChild(resolved.cleanPath, self.contextDir.cleanPath)
+   }
    let transfer = try path.buildTransfer(id: packet.id, contextDir: self.contextDir, complete: true)
```

이 한 패치가 세 가지를 동시에 막습니다.

1. **심링크 유출** — `resolvingSymlinksInPath()`가 실제 타겟까지 따라가 검사
2. **절대 경로 요청** — 가드가 I/O 이전으로 이동해 호스트 측 읽기 자체가 발생하지 않음
3. **percent-decoding fallback** — 재조립된 경로도 동일한 가드를 통과해야 함

`walk()`가 원래 하고 있던 일을 `read()`/`info()`가 뒤늦게 따라잡은 셈이죠.

사용자 입장에서의 조치는 간단합니다. **`container`를 1.2.0 이상으로 업데이트**하시길 권장드립니다. 그 전까지는 신뢰할 수 없는 저장소를 빌드 컨텍스트로 쓰지 않는 것 말고는 마땅한 완화책이 없습니다.

***

# 5. 마치며

이번 건은 화려한 메모리 손상 없이도 **"검사를 언제, 무엇에 대해 하느냐"** 한 가지만 어긋나면 신뢰 경계가 통째로 무너질 수 있다는 걸 보여준 사례였습니다. 정리하면 세 가지가 남습니다.

- **같은 파일 안의 비대칭은 가장 좋은 취약점 냄새입니다.** `walk()`는 `resolvingSymlinksInPath()`로 재검사하는데 `read()`/`info()`는 하지 않았죠. 같은 액터, 같은 컨텍스트, 같은 신뢰 경계인데 한쪽에만 방어가 있다면 그건 설계 의도가 아니라 **누락**일 가능성이 큽니다.
- **경로 컨테인먼트는 문자열이 아니라 파일에 대해 검사해야 합니다.** `fs_components`처럼 `..`를 텍스트로 접는 정규화는 "이름이 안전해 보이는지"까지만 보장해 줍니다. `realpath` / `resolvingSymlinksInPath` / `openat(O_NOFOLLOW)`처럼 **커널이 해석한 결과**를 기준으로, 그것도 **I/O 이전에** 검사해야 합니다.
- **신뢰 경계의 방향을 먼저 의심해 보시길 권합니다.** "게스트는 호스트로부터 격리되어 있다"는 직관 때문에, 정작 호스트가 게스트의 요청을 받아 파일을 서빙하는 이 구조는 오래 검토되지 않았을 가능성이 큽니다. VM 격리 모델을 감사하실 때는 **어느 쪽이 서버인지**부터 그려 보시면 좋겠습니다.

그리고 마지막으로 — 서두에 적었듯 이 건은 **중복 제보**였습니다. 저 말고도 여러 명이 같은 결함을 보고 있었고, 제가 첫 제보자도 아니었어요. 그럼에도 Apple이 중복 제보자까지 크레딧에 함께 올려준 덕분에 어드바이저리에 `blackcon`이라는 이름을 남길 수 있었습니다. 같은 코드를 여러 사람이 동시에 들여다보고 있다는 사실 자체가 이 생태계가 건강하게 굴러가고 있다는 신호라고 생각하니, 아쉬움보다는 반가움이 조금 더 컸던 것 같습니다.

읽어주셔서 감사합니다! 🙏

***

# 6. 참고 링크 (References)

**보안 권고문**
- GHSA-2v2q-4q35-h585 — *Build filesystem sync discloses host files outside the build context via symlinks*: <https://github.com/apple/container/security/advisories/GHSA-2v2q-4q35-h585>

**취약 코드 (main 기준 현재 상태 — 이미 패치됨)**
- 빌드 파일 동기화: [`Sources/ContainerBuild/BuildFSSync.swift`](https://github.com/apple/container/blob/main/Sources/ContainerBuild/BuildFSSync.swift)
- 경로 컨테인먼트 헬퍼: [`Sources/ContainerBuild/URL+Extensions.swift`](https://github.com/apple/container/blob/main/Sources/ContainerBuild/URL+Extensions.swift)

**제품 저장소 / 다운로드**
- apple/container (GitHub): <https://github.com/apple/container>
- 릴리스(1.2.0 이상 권장): <https://github.com/apple/container/releases>

**관련 CWE**
- CWE-22: Improper Limitation of a Pathname to a Restricted Directory: <https://cwe.mitre.org/data/definitions/22.html>
- CWE-59: Improper Link Resolution Before File Access ('Link Following'): <https://cwe.mitre.org/data/definitions/59.html>
