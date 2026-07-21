---
title: "Claude Code의 /btw와 fork는 정말 Fable을 쓸까? — 모델 라우팅 추적기"
categories: [Research, AI-Security]
tags: [Claude, Claude-Code, Fable5, Opus, model-routing, refusal-fallback, LLM, AI, security, reverse-engineering, btw, fork, jailbreak]
date: 2026-07-21 09:00:00 +0900
description: "x.com에서 돌던 소문 — /btw와 fork를 쓰면 Fable 5를 계속 쓸 수 있다. 정말일까? 세션 로그와 바이너리를 파헤쳐 Claude Code가 Fable과 Opus 사이를 어떻게 오가는지(refusal-fallback, model latch, visible/silent lane) 추적한 기록입니다. 결론부터 말하면, 그 소문은 사실이 아니었습니다."
pin: false
---

안녕하세요! 이번 글에서는 지난주 x.com에서 잠깐 돌았던 한 가지 "꿀팁"을 검증해 본 이야기를 풀어보려 합니다.

소문은 이랬어요.

> Claude Code에서 **`/btw`로 질문하고, 그걸 fork해서 대화를 이어가면 Fable 5를 계속 쓸 수 있다.**

Fable 5는 Mythos급 성능인데 가드레일이 워낙 빡빡해서 보안 관련 질문은 자주 막힙니다([예전 글](/posts/fable5-jailbreak/) 참고). 그러니 "btw/fork로 Fable을 안정적으로 붙잡아 둘 수 있다"는 얘기는 솔깃할 수밖에 없죠. 그런데 직접 테스트해 보니 — **저는 Fable을 붙잡는 데 실패했습니다.** 😭

왜 실패했는지 궁금해서 세션 로그와 바이너리를 파고들었고, 그 과정에서 Claude Code가 Fable과 Opus 사이를 오가는 꽤 흥미로운 메커니즘을 발견했습니다. 오늘은 그 추적 후기를 남겨보려 합니다.

> 이 글의 모든 관측은 제 로컬 환경(Claude Code v2.1.215/216)과 제 세션 로그를 대상으로 한 것입니다. 안전장치를 뚫는 방법이 아니라, "내가 지금 실제로 어떤 모델에 요청을 보내고 있는가"를 추적한 투명성 분석에 가깝습니다.
{: .prompt-info }

***

# 1. `/btw`에는 두 얼굴이 있다

먼저 `/btw`가 뭔지부터. 이건 메인 작업을 방해하지 않고 "잠깐 옆에서" 질문을 던지는 side-question 기능입니다. 바이너리를 뜯어보면 `/btw`는 **서로 다른 두 개의 코드 경로**로 갈립니다.

1. **빌트인 side-question** (`runSideQuestion`) — 도구 없음, 단발 응답, 그리고 결정적으로 **`skipTranscript: true`** 라 로그에 아무것도 안 남습니다.
2. **Fork 워커** (`subagent_type:"fork"`) — 전체 도구, 최대 200턴, transcript 기록. 기능 플래그(`CLAUDE_CODE_FORK_SUBAGENT` 등)로 게이팅됩니다.

두 경로가 모델에게 실제로 **주입하는 프롬프트**부터 성격이 정반대입니다. 먼저 **side-question**은 이런 `<system-reminder>`를 붙여요(바이너리 원문).

```text
<system-reminder>This is a side question from the user. You must answer this question directly in a single response.
IMPORTANT CONTEXT:
- You are a separate, lightweight agent spawned to answer this one question
- The main agent is NOT interrupted - it continues working independently in the background
- You share the conversation context but are a completely separate instance
- Do NOT reference being interrupted or what you were "previously doing" - that framing is incorrect
CRITICAL CONSTRAINTS:
- You have NO tools available - you cannot read files, run commands, search, or take any actions
- This is a one-off response - there will be no follow-up turns
- You can ONLY provide information based on what you already know from the conversation context
- NEVER say things like "Let me try...", "I'll now...", "Let me check...", or promise to take any action
- If you don't know the answer, say so - do not offer to look it up or investigate
Simply answer the question with the information you have.</system-reminder>
```

반면 **fork 워커**는 부모의 transcript를 통째로 상속시키면서 이런 `<fork-boilerplate>`를 붙입니다(스톡 원문).

```text
<fork-boilerplate>
You are a worker fork. The transcript above is the parent's history — inherited reference, not your situation. You are NOT a continuation of that agent. Execute ONE directive, then stop.
Hard rules:
- Do NOT spawn subagents with the <Task> tool. The "default to forking" guidance is for the parent; you ARE the fork, execute directly.
- One shot: report once and stop. No follow-up questions, no proposed next steps, no waiting for the user.
Guidelines (your directive may override any of these):
- Stay in scope. Other forks may be handling adjacent work; if you spot something outside your directive, note it in a sentence and move on.
- Open with one line restating your task, so the parent can spot scope drift at a glance.
- Be concise — as short as the answer allows, no shorter. Plain text, no preamble, no meta-commentary.
- If you committed changes, list the paths and commit hashes in your report.
</fork-boilerplate>
Your directive: <지시문>
```

한쪽은 "도구 없이, 단발로, 아는 것만" 답하라 하고, 다른 쪽은 "부모 컨텍스트를 물려받아 지시 하나를 끝까지 실행"하라 합니다. 이름은 같은 `/btw`인데 속은 딴판이죠.

즉 소문의 "btw"는 ①, "fork"는 ②를 가리키는 셈이죠. 그리고 여기서 첫 번째 벽에 부딪힙니다.

***

# 2. 단일 `/btw`는 검증조차 안 된다

"btw가 Fable로 답했다"는 걸 확인하려면 로그의 `model` 필드를 봐야 합니다. 그런데 side-question 경로는 `skipTranscript: true`로 동작해서 `*.jsonl`에 **아무 흔적도 남기지 않아요.** 응답은 인메모리 히스토리에만 잠깐 있다가 프로세스가 죽으면 사라집니다.

```
promptMessages: [...],
querySource: "side_question",
maxTurns: 1,
skipCacheWrite: !0,
skipTranscript: !0,   // ← 여기
```

그러니 단일 `/btw`가 정말 Fable이었는지는 **사후에 알 방법이 없습니다.** 그냥 "Fable이었길" 기도하는 수밖에요. 🙏

이게 답답해서, 저는 제 로컬 바이너리에서 이 `skipTranscript:!0`를 `!1`로 뒤집는 1바이트 패치를 만들었습니다(동일 길이 교체라 재서명만 하면 실행됩니다). 이제 side-question도 로그에 남으니, 드디어 "실제로 어떤 모델이 답하는지"를 볼 수 있게 됐죠.

> 참고로 저는 이 실험을 하며 위 §1의 fork-boilerplate 리마인더도 `Full-capability fork…` 변형으로 바꿔둔 상태였습니다. 그래서 뒤에 나오는 스크린샷의 fork-boilerplate 텍스트가 §1의 스톡 원문과 다르게 보일 거예요. 다만 이 글의 핵심인 **모델 라우팅(Fable↔Opus)은 프롬프트 텍스트가 아니라 refusal-fallback으로 결정**되므로, 이 변형은 분석 결론에는 영향을 주지 않습니다.
{: .prompt-info }

> 참고로 이 인스턴스는 fork 실험이 켜져 있어서, `/btw`가 side-question이 아니라 **로깅되는 서브에이전트**(`aside_question`, `afork`)로 스폰됩니다. 덕분에 패치 없이도 상당 부분 관측이 가능했어요. (이건 인스턴스 설정에 따라 다릅니다.)
{: .prompt-info }

***

# 3. 첫 충격: 메인은 Fable인데 btw는 Opus

로그를 열자마자 이상한 걸 봤습니다. **메인 세션 모델은 분명 Fable 5인데, btw로 던진 질문의 응답은 Opus-4-8로 찍혀 있었어요.**

![같은 세션에서 /btw 응답은 OPUS, fork 이후 응답은 FABLE로 갈린다 (usage.py 실시간 추적)](/posts/claude-code-btw-fork-model-routing/btw-opus-fork-fable.png)
_같은 세션·같은 질문(수학 난제)인데, `/btw`에 대한 응답(위 박스)은 **OPUS**, fork 이후 응답(아래 박스)은 **FABLE**로 갈린다. `usage.py`로 세션 로그의 `model` 필드를 실시간 추적한 화면._

같은 세션, 같은 무해한 질문("수학 난제 예시를 알려줘")을 놓고 실측한 결과:

| 시각 | 주체 | 모델 |
|---|---|---|
| side_question (btw) | `aside_question` | **claude-opus-4-8** |
| fork | `afork` | **claude-fable-5** |

같은 세션 안에서 btw는 Opus, fork는 Fable로 갈렸습니다. "어? fork는 Fable이 맞네?" 싶다가도, btw가 왜 Opus인지가 도무지 이해가 안 됐죠. 메인이 Fable인데 말이에요.

***

# 4. 모델의 거짓말 — "저는 Fable 5입니다"

더 황당한 건 이겁니다. fork에게 **"너 무슨 모델이야?"** 라고 물었더니 이렇게 답했어요.

> 현재 모델: **Claude Fable 5** (모델 ID: `claude-fable-5`)

그런데 그 턴의 raw 로그를 열어보니:

```
agent: fork | ts: 2026-07-20T05:45:09Z
SERVED model -> claude-opus-4-8
text: 현재 모델: **Claude Fable 5** (모델 ID: claude-fable-5)
```

서버가 기록한 실제 모델은 **Opus 4.8**. 모델은 자기 가중치를 들여다보지 못하고, 프롬프트가 "너는 Fable"이라고 하니 그렇게 답할 뿐이었습니다. **모델의 자기소개는 신뢰할 수 없고, 진실은 언제나 로그의 `model` 필드에 있습니다.**

fork만 그런 게 아니에요. 단발 `/btw`로 물어봐도 똑같습니다.

![/btw로 물으면 'Fable 5'라 답하지만 usage.py의 서버 model은 OPUS다](/posts/claude-code-btw-fork-model-routing/btw-claims-fable-served-opus.png)
_`/btw 현재 사용중인 모델을 알려줘`에 대한 답은 당당하게 "Fable 5 (`claude-fable-5`)"(위). 하지만 같은 응답을 `usage.py`로 추적하면 서버가 실제로 서빙한 모델은 **OPUS**(아래). 모델의 말과 실측 로그가 정면으로 어긋난다._

> 교훈: LLM에게 "너 무슨 모델이야?"라고 물어서 나온 답은 근거가 아닙니다. self-description이 아니라 실측 로그(서버가 되돌려준 `model`)를 보세요.
{: .prompt-warning }

***

# 5. 바이너리 해부: refusal-fallback과 latch

여기서부터 바이너리를 본격적으로 팠습니다. 핵심은 **거절 시 실행되는 유효 모델 결정 함수**였어요.

```js
function k3(){ let e, t = Pv();          // Pv() = 메인 루프 모델 override
  if (t !== void 0) e = t;                // ← override가 있으면 그 모델을 사용
  else { let r = G8(); e = ... } }        //   아니면 정상 선택 모델
}
```

그리고 이 override를 설정/조회/해제하는 상태들:

- `latchRefusalFallbackModel` — fallback 모델을 **latch(고정)**
- `getRefusalFallbackModelLatch` / `clearRefusalFallbackModelLatch`
- 요청에 붙는 헤더: `x-is-refusal-fallback`, `x-cc-fallback-latched-by`, `x-cc-fallback-from-model` …

정리하면 이렇습니다. Fable의 safeguard가 어떤 메시지를 flag하면 → 거절(refusal) → **refusal-fallback**이 발동해 fallback 모델(Opus)을 **세션 override로 latch**합니다. 한 번 걸리면 유효 모델(`k3()`)은 계속 Opus를 반환하죠.

**결정적으로, 이 fallback 결정은 HTTP 헤더로만 오가고 transcript에는 기록되지 않습니다.** 그래서 btw 로그에는 "Fable을 시도했다 거절당했다"는 흔적 없이, **Opus 결과만** 툭 남습니다. 우리가 4장에서 본 그 미스터리의 정체가 이거예요.

***

# 6. 왜 메인은 알려주고 btw는 조용할까 — visible vs silent lane

그런데 이상하죠. 제가 **메인 세션**에서 "수학 문제를 알려줘"라고 했을 땐 이런 에러가 떴거든요.

```
API Error: Fable 5's safeguards flagged this message ...
Claude Code can't respond to this request with Fable 5.
Double press esc to edit your last message, or try a different model with /model.
```

때로는 아예 멈추고 **사용자에게 선택을 묻기도** 합니다.

![메인 세션에서 Fable safeguard가 걸리면 Opus로 바꿀지 사용자에게 묻는 다이얼로그](/posts/claude-code-btw-fork-model-routing/main-safeguard-switch-dialog.png)
_메인 세션이 flag되면 `Session paused`와 함께 **"1. Switch to Opus 4.8 / 2. Edit prompt and retry with Fable 5"** 를 사용자에게 묻는다. 이게 visible lane의 전형 — 전환을 숨기지 않고 명시적으로 알린다. btw·fork는 이 다이얼로그 없이 조용히 Opus로 넘어간다._

메인에선 이렇게 **대놓고 알려주는데**(하드 정지든, 위처럼 전환을 묻는 다이얼로그든), btw/fork에선 아무 말 없이 조용히 Opus로 넘어갑니다. 왜 다를까요?

바이너리에서 답을 찾았습니다. 거절 처리에는 **세 갈래**가 있고, 그걸 "lane"이 결정합니다.

```js
refusalFallbackModel     = !x ? (Ns ?? ef) : void 0;
refusalFallbackModelLane = !x ? (Ns!==void0 ? "visible" : ef!==void0 ? "silent" : void0) : void 0;
```

- **fallback 모델이 아예 없음(`x`)** → `model_refusal_no_fallback` = **하드 정지 + 알림** (제가 본 그 에러)
- **`Ns`(visible lane) 설정** → 자동 전환 + footprint (메인)
- **`ef`(silent lane) 설정** → **조용히 전환** (btw/fork)

그리고 로그로 이걸 교차 검증했습니다. `model_refusal_*` 이벤트가 **어디서** 발생하는지 세어봤더니:

| 이벤트 | main | side_question | fork |
|---|---|---|---|
| `model_refusal_fallback` (전환 footprint) | 53 | 0 | 0 |
| `model_refusal_no_fallback` (하드 정지 경고) | 18 | 0 | 0 |

refusal 이벤트는 **오직 메인에서만** 나오고, btw/fork에선 **단 한 건도** 없습니다. 그런데도 결과는 Opus로 서빙됐죠 = **알림 없이 조용히 fallback**됐다는 증거입니다.

> 한마디로: **메인은 "모델 바꿉니다"라고 사용자에게 알려주지만, btw/fork는 그냥 몰래 바꿉니다.** 이게 "btw로 Fable 쓴다"는 착각이 생긴 지점이기도 해요 — 사용자는 전환을 못 보니까요.
{: .prompt-warning }

***

# 7. 무너진 가설들

여기까지 오면 "그럼 btw는 원래 Opus로 라우팅되는구나" 싶지만, 데이터를 더 모으자 제 가설들이 하나씩 무너졌습니다. 이게 이번 분석에서 제일 재밌었던 부분이에요.

**가설 ① "btw(side_question)는 항상 Opus다"** → ❌ 반증.
Fable로 서빙된 side_question이 실제로 존재했습니다. 첫 턴 기준 Fable 비율은 side_question 25%(3/12), fork 44%(4/9). 둘 다 Fable로 답한 적이 있어요.

**가설 ② "side_question은 컨텍스트가 무거워서 더 자주 flag된다"** → ❌ 반증.
같은 세션에서 입력 토큰을 재봤더니:

| 세션 | side_q 입력 | fork 입력 | 결과 |
|---|---|---|---|
| A | 24,716 tok → **Opus** | 32,859 tok → **Fable** |
| B | 24,628 tok → **Opus** | 32,673 tok → **Fable** |

**fork가 오히려 컨텍스트가 더 큰데도 Fable을 유지**했습니다. 크기 가설은 틀렸어요.

**가설 ③ "보안 주제면 Opus, 무해하면 Fable"** → ❌ 약함.
무해 질문 Opus12/Fable6(33%), 보안 질문 Opus2/Fable1(33%)로 Fable 비율이 거의 같았습니다. 주제가 모델을 깔끔하게 가르지 못했어요.

그럼 뭐가 모델을 정하냐 — 남은 답은 하나였습니다. **세션의 latch 상태.** 한 번 fallback이 걸리면 그 세션/스레드에선 이후 무해한 질문도 대체로 Opus로 나옵니다. 주제도 크기도 아니고, "이 세션이 이미 넘어갔는가"가 지배 요인이었죠.

***

# 8. 그래서 Fable은 언제 돌아오나

latch가 걸리면 영원히 Opus일까요? 스레드 내 모델 전환을 세어봤습니다.

- Fable → Opus: **19건**
- Opus → Fable(자동 복귀): **2건**

대체로 단조(한 번 Opus면 유지)지만, **드물게 스스로 Fable로 복귀**하기도 합니다. 바이너리에 `refusalFallbackSilentRearm`(silent 재무장)이 있어서, 가끔 Fable을 다시 시도하게 해요. 하지만 신뢰할 수 있는 복구법은 딱 하나입니다.

> **`/model`로 모델을 다시 선택하면 latch가 해제됩니다.** 바이너리에서 `clearRefusalFallbackModelLatch`가 `/model` 변경(`tengu_config_model_changed`) 시 호출되는 걸 확인했어요.
{: .prompt-tip }

***

# 9. 결론

정리하면, 소문 "btw/fork로 Fable을 쓸 수 있다"는 **사실이 아니었습니다.** 오히려 그 반대에 가까웠어요.

- **btw(side_question)와 fork는 둘 다 Fable을 시도하다, 요청이 safeguard에 걸리면 조용히 Opus-4-8로 fallback**합니다.
- 무엇으로 서빙되느냐의 지배 요인은 **주제도, 컨텍스트 크기도 아니라 "세션의 refusal-fallback latch 상태"** 였습니다.
- **메인은 전환을 알려주지만(visible lane), btw/fork는 조용히 넘어갑니다(silent lane).** "btw로 Fable 쓴다"는 착각은 이 무알림 전환에서 비롯된 것으로 보입니다.
- 보안 연구처럼 Fable이 자주 flag되는 맥락에선, btw/fork로도 **Fable을 안정적으로 붙잡을 수 없습니다.** 결국 Opus로 밀려나요.
- 그리고 무엇보다 — **btw·fork는 안전장치를 우회하는 통로가 아닙니다.** 컨텍스트·도구·턴 수·알림 방식만 바뀔 뿐, 모델과 API 안전 계층은 동일하게 적용됩니다. 오히려 flag되면 더 보수적인 fallback으로 넘어가죠.

이번 검증에서 제일 크게 배운 건, **"모델에게 물어보지 말고 로그를 보라"** 였습니다. fork가 자신 있게 "저는 Fable입니다"라고 하던 그 순간에도, 실제로 토큰을 만들어낸 건 Opus였으니까요. self-description은 프롬프트의 메아리일 뿐, 진실은 언제나 `model` 필드에 있습니다.

읽어주셔서 감사합니다. 다음 글에서 또 만나요! 👋

> 이 분석은 v2.1.215/216 기준이며, 관측은 fork 실험이 켜진 특정 인스턴스 한정입니다. CLI 버전이나 설정이 바뀌면 세부 동작은 달라질 수 있습니다.
{: .prompt-info }
