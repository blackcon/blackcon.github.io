---
title: 코드 생성과 토론을 위한 Code Llama | 새로운 대화형 언어 모델 소개
categories: [AI, llama]
tags: [AI, llama, meta]
date: 2023-08-28 09:08:00 +0900
---

# 추천 글
- [Meta에서 출시한 인공지능 LLaMA를 사용해보자 (ChatGPT만큼 되나?)](https://blackcon.github.io/posts/how-to-use-llama/)
- [LLaMA.cpp를 기반으로 한 코딩 Asistant 프로젝트 (Copilot 대체하기)](https://blackcon.github.io/posts/how-to-set-local-ai-for-coding/)

# Code LLaMA 공개
[Meta AI](https://ai.meta.com/)가 선보인 새로운 대화형 언어 모델인 [Code Llama](https://ai.meta.com/blog/code-llama-large-language-model-coding/)에 대해 이야기해보려 합니다. 이 모델은 텍스트 프롬프트를 활용하여 코드를 생성하고 토론할 수 있는 대용량 언어 모델(Large Language Model, LLM)입니다. 

![code-llama.gif](/posts/2023-08-28-code-llama.gif)

[Code Llama](https://ai.meta.com/blog/code-llama-large-language-model-coding/)는 [LLaMA2 모델](https://ai.meta.com/llama/)을 기반으로 하여 생성되었으며, 코딩 작업에 있어서 현재까지 공개적으로 이용 가능한 LLM 중 가장 최신 기술을 적용한 모델로, 개발자들의 작업을 효율적으로 만들어주고 코딩을 배우는 사람들에게도 도움을 줄 수 있을 듯합니다.

# Code Llama 성능 비교
1. 테스트 방법
    - Code Llama의 성능 평가를 위해 두 가지 인기 코딩 벤치마크인 **HumanEval** 과 **Mostly Basic Python Programming (MBPP)** 을 사용함.
    - 정의
        - **HumanEval**: 모델이 docstrings을 기반으로 코드 완성을 수행하는 능력을 테스트
        - **MBPP**: 모델이 설명을 기반으로 코드 작성을 수행하는 능력을 테스트함.
2. 테스트 결과
    - Code Llama는 오픈 소스 코드 전용 LLM보다 더 나은 성능을 보이며, Llama2보다도 우수한 성과를 달성함. (아직 GPT4 보다는 낮네요 ㅠㅠ)
    - Code Llama 34B 모델은 HumanEval에서 53.7%, MBPP에서 56.2%의 점수를 기록함. 
    - 이는 다른 최첨단 오픈 솔루션과 비교하여 가장 높은 점수를 달성

3. Performance Table

![performance.jpg](/posts/2023-08-28-code-llama-performance.jpg)

# Code Llama의 맛보기 
Code Llama는 개발자들의 작업을 보다 효율적으로 만들어주며, 코딩을 배우려는 입문자들의 진입 장벽을 낮추는 데 도움을 줄 수 있습니다. 이 모델은 코드 생성 및 코드 관련 자연어 생성에 특화되어 있어서, 코드와 자연어 프롬프트로부터 코드 및 자연어에 대한 코드를 생성할 수 있습니다. 

예를 들어, "피보나치 수열을 출력하는 함수를 작성하세요"와 같은 프롬프트로부터 코드를 생성할 수 있을 뿐만 아니라, 코드 완성 및 디버깅과 같은 작업에도 활용될 수 있습니다. Python, C++, Java, PHP, TypeScript (JavaScript), C#, Bash 등 오늘날 가장 인기 있는 프로그래밍 언어를 지원한다는 점도 주목할 만합니다.
- Demo: https://labs.perplexity.ai/
- 결과
    ![2023-08-28-code-llama-sample.png](/posts/2023-08-28-code-llama-sample.png)

# 모델 종류 
Code Llama는 7B, 13B 및 34B 파라미터 크기의 세 가지 다양한 모델을 공하였구요. 각 모델은 500B 토큰의 코드 및 코드 관련 데이터로 훈련되었으며, 이 중 7B와 13B 모델은 FIM 기능을 포함하여 코드 완성 작업을 지원했다고 합니다. 파라미터 크기가 작은 모델은 낮은 대기 시간이 필요한 실시간 코드 완성과 같은 작업에 적합하며, 34B 모델은 더 나은 코딩 지원을 제공합니다.

또한 Code Llama의 두 가지 특화된 변형도 소개되었습니다. 
1) **Code Llama - Python**
    - Code Llama의 언어 특화 변형으로, Python 코드의 100억 토큰에 더 세밀하게 조정되었습니다. 
    - Python은 코드 생성에 가장 많이 벤치마킹된 언어이며, Python과 PyTorch가 AI 커뮤니티에서 중요한 역할
2) **Code Llama - Instruct**
    - 지시어(Instruct)를 통해 훈련된 변형으로
    - 프롬프트의 사용자가 기대하는 결과를 더 잘 이해하고 도움과 안전한 답변을 생성할 수 있도록 조정됨

# 끝으로
다음에는 Code Llama 모델을 이용하여 Local Machine에 셋팅하는 방법을 소개드리겠습니다. 전반적인 골조는 이전에 작성한 "[LLaMA.cpp를 기반으로 한 코딩 Asistant 프로젝트 (Copilot 대체하기)](https://blackcon.github.io/posts/how-to-set-local-ai-for-coding/)"와 비슷할 듯 하구요. 포스팅 글이 올라오기 전에 Coding Asistant를 사용해보고 싶으시다면, [이전 글](https://blackcon.github.io/posts/how-to-set-local-ai-for-coding/)을 참고해주세요 :)

# 참고
- [Code Llama Blog Post](https://ai.meta.com/blog/code-llama-large-language-model-coding/)
- [Code Llama Research Paper](https://ai.meta.com/research/publications/code-llama-open-foundation-models-for-code/)
- [Code Llma | github](https://github.com/facebookresearch/codellama)
- [Code Llama 모델 다운로드](https://ai.meta.com/resources/models-and-libraries/llama-downloads/)