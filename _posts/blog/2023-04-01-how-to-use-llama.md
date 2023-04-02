---
title: Meta에서 출시한 인공지능 LLaMA를 사용해보자 (ChatGPT만큼 되나?)
categories: [AI, llama]
tags: [AI, llama, meta]
date: 2023-04-01 09:08:00 +0900
---
_**참고**: 인공지능을 전혀 공부하지 않은 사람이 작성하는 글입니다. 사용하는 단어나 문장에 대해서 틀릴 수 있으니, 수정이 필요할 경우 언제든 댓글 부탁드립니다._

# LLaMA 
## 소개
&nbsp; [Meta](https://www.meta.com/)에서는 대규모 언어 모델인 [LLaMA(Large Language Model Meta AI)](https://ai.facebook.com/blog/large-language-model-llama-meta-ai/)를 공개했다. 보통의 인공지능 모델들은 빵빵(?)한 서버 환경이 뒷 받침 되어야 하는데, LLaMA의 경우는 일반 사용자의 PC에서도 수행이 가능할 정도로 개발이 잘 되어 있습니다. <br>
제가 본 기사 중 일부의 내용으로 보아하니 다른 AI와 견주어도 뒤지지 않을 만큼의 성능을 지닌 AI로 보여집니다! 셋팅을 하면서 어떤게 또 가능할지, 관련된 포스팅들을 틈틈히 써볼 수 있도록 할게요. 보다 자세한 내용은 [관련 기사](https://www.aitimes.com/news/articleView.html?idxno=149681)를 참고길 바랍니다.

## 기사 중 일부 발췌([AI타임스](https://www.aitimes.com/news/articleView.html?idxno=149681))
> 기본형인 66B(매개변수 650억개)를 비롯해 ▲7B(70억개) ▲13B(130억개) ▲33B(330억개) 등 용도에 맞춰 리소스를 줄일 수 있도록 다양한 버전을 출시할 계획이다.
> 
> 오픈AI의 'GPT-3.0'와 'GPT-3.5(챗GPT)'는 매개변수가 1750억개, 구글의 'PaLM'은 5400억개에 달한다. 
> 
> 하지만 메타는 매개변수를 키우기보다 LLM 훈련에 사용하는 토큰(텍스트 데이터 단위)의 양을 늘여, 품질을 높였다고 설명했다. LLaMA 65B와 33B에 1조4000억개, 가장 작은 모델인 LLaMA 7B의 경우에도 1조개를 사용했다고 밝혔는데, 이는 구글의 딥마인드가 지난해 9월 공개한 최근 LLM '친칠라'의 1조4000억개와 같은 수준이다.

# LLaMA.cpp
![llama.cpp](https://user-images.githubusercontent.com/1991296/227761327-6d83e30e-2200-41a6-bfbb-f575231c54f4.png)
## 소개
python으로 작성된 LLaMA가 공개된 후 C/C++ 을 베이스로 실행할 수 있는 [프로젝트](https://github.com/ggerganov/llama.cpp)가 공개되었습니다. 하여 이 글에서는 `LLaMA.cpp`를 베이스로 사용하는 방법을 적어보도록 하겠습니다. 이유는... 인공지능 환경 셋팅이 낯선 저에게는 기본 LLaMA 보다, `LLaMA.cpp` 의 옵션이 더 간결하고 쉬웠으며 에제도 다양하게 있어서입니다 :) <br>

그럼 아래의 순서대로 `LLaMA.cpp`를 셋팅해보고 다양한 예제들을 실행해보도록 하겠습니다.
```
1) git clone
2) 소스코드 컴파일
3) 모델 데이터 다운로드
4) 모델 데이터 변환 (`*.pth` to to `ggml FP16` format)
5) 예제 코드 실행
```

## 실습하기
### 1) 프로젝트 클론(clone)하기
환경 셋팅을 위해서는 프로젝트를 Local Machine으로 받아와야 합니다. 간단하게 `git clone`으로 프로젝트를 클론해봅니다.
```bash
➜  git clone https://github.com/ggerganov/llama.cpp.git
Cloning into 'llama.cpp'...
remote: Enumerating objects: 1246, done.
remote: Counting objects: 100% (508/508), done.
remote: Compressing objects: 100% (102/102), done.
remote: Total 1246 (delta 445), reused 422 (delta 405), pack-reused 738
Receiving objects: 100% (1246/1246), 1.28 MiB | 3.16 MiB/s, done.
Resolving deltas: 100% (758/758), done.
```

### 2) 소스코드 컴파일
프로젝트에서 제공하는 소스코드와 Makefile을 신뢰하고 바로 컴파일을 시켜보겠습니다. 별다른 에러 없이 `====  Run ./main -h for help.  ====`까지 출력된다면 컴파일 성공입니다 :)
```bash
➜ make -j
I llama.cpp build info:
I UNAME_S:  Darwin
I UNAME_P:  arm
I UNAME_M:  arm64
I CFLAGS:   -I.              -O3 -DNDEBUG -std=c11   -fPIC -Wall -Wextra -Wpedantic -Wcast-qual -Wdouble-promotion -Wshadow -Wstrict-prototypes -Wpointer-arith -Wno-unused-function -pthread -DGGML_USE_ACCELERATE
I CXXFLAGS: -I. -I./examples -O3 -DNDEBUG -std=c++11 -fPIC -Wall -Wextra -Wpedantic -Wcast-qual -Wno-unused-function -pthread
I LDFLAGS:   -framework Accelerate
I CC:       Apple clang version 14.0.0 (clang-1400.0.29.202)
I CXX:      Apple clang version 14.0.0 (clang-1400.0.29.202)

cc  -I.              -O3 -DNDEBUG -std=c11   -fPIC -Wall -Wextra -Wpedantic -Wcast-qual -Wdouble-promotion -Wshadow -Wstrict-prototypes -Wpointer-arith -Wno-unused-function -pthread -DGGML_USE_ACCELERATE   -c ggml.c -o ggml.o
c++ -I. -I./examples -O3 -DNDEBUG -std=c++11 -fPIC -Wall -Wextra -Wpedantic -Wcast-qual -Wno-unused-function -pthread -c llama.cpp -o llama.o
c++ -I. -I./examples -O3 -DNDEBUG -std=c++11 -fPIC -Wall -Wextra -Wpedantic -Wcast-qual -Wno-unused-function -pthread -c examples/common.cpp -o common.o
c++ -I. -I./examples -O3 -DNDEBUG -std=c++11 -fPIC -Wall -Wextra -Wpedantic -Wcast-qual -Wno-unused-function -pthread examples/main/main.cpp ggml.o llama.o common.o -o main  -framework Accelerate
c++ -I. -I./examples -O3 -DNDEBUG -std=c++11 -fPIC -Wall -Wextra -Wpedantic -Wcast-qual -Wno-unused-function -pthread examples/quantize/quantize.cpp ggml.o llama.o -o quantize  -framework Accelerate
c++ -I. -I./examples -O3 -DNDEBUG -std=c++11 -fPIC -Wall -Wextra -Wpedantic -Wcast-qual -Wno-unused-function -pthread examples/perplexity/perplexity.cpp ggml.o llama.o common.o -o perplexity  -framework Accelerate
c++ -I. -I./examples -O3 -DNDEBUG -std=c++11 -fPIC -Wall -Wextra -Wpedantic -Wcast-qual -Wno-unused-function -pthread examples/embedding/embedding.cpp ggml.o llama.o common.o -o embedding  -framework Accelerate

====  Run ./main -h for help.  ====
```
### 3) Model weight 다운로드 받기
LLaMA 프로젝트는 소스코드만 공개하고 `model`은 공개하지 않았는데요. 해당 [구글 Form](https://forms.gle/jk851eBVbX1m5TAv5)에 신청을 하면 모델 데이터를 받을 수 있다고 하네요. 또는 누군가가 이 [PR](https://github.com/facebookresearch/llama/pull/73/files)을 통해서 토렌트 주소를 공개를 해뒀다고 하는데 참고(?) 하세요.

**참고**: 모델 데이터가 없는 상태에서 예제 코드를 실행해보았습니다. 아래와 같이 `error: failed to load model` 라며 모델 데이터 로딩에 실패를 하게 되네요.
```bash
➜  llama.cpp git:(master) ./examples/chat.sh
main: seed = 1680252220
llama_model_load: loading model from './models/7B/ggml-model-q4_0.bin' - please wait ...
llama_model_load: failed to open './models/7B/ggml-model-q4_0.bin'
llama_init_from_file: failed to load model
main: error: failed to load model './models/7B/ggml-model-q4_0.bin'

➜  llama.cpp git:(master) ll models
total 848
-rw-r--r--  1 user  staff   422K  3 31 17:43 ggml-vocab.bin
```
### 4) Model format 변환하기

```bash
# install Python dependencies
python3 -m pip install torch numpy sentencepiece

# convert the 7B model to ggml FP16 format
python3 convert-pth-to-ggml.py models/7B/ 1

# quantize the model to 4-bits (using method 2 = q4_0)
./quantize ./models/7B/ggml-model-f16.bin ./models/7B/ggml-model-q4_0.bin 2

# run the inference
./main -m ./models/7B/ggml-model-q4_0.bin -n 128
```
### 5) 예제 실행하기
해당 소스코드에서 제공하는 예제는 다양하게 있는데요. 이 중에서도 챗봇을 한 번 실행토록 해보겠습니다. 나름 잘 되는 듯 하네요.
![chatllama-chat13B.png](/posts/chatllama-chat13B-2.png)
이 예제코드를 응용해서 가상의 `Warren Buffett`을 만들어보고자 합니다. Warren에 대한 정보는 [Wiki](https://en.wikipedia.org/wiki/Warren_Buffett)에서 긁어 왔으며, 이 데이터를 LLaMA에게 주입하고 `You are Warren`이라고 정의랠 해보았습니다. 이런 셋팅을 완료한 수 몇가지 질문을 하니,,, Warren과 비슷한 답변을 하는 너낌이 드네요 :)
- bash script (`chat-13b-warren.sh`)
  ```bash
  #!/bin/bash

  cd "$(dirname "$0")/.." || exit

  MODEL="${MODEL:-./models/13B/ggml-model-q4_0.bin}"
  USER_NAME="${USER_NAME:-User}"
  AI_NAME="${AI_NAME:-Warren}"

  # Adjust to the number of CPU cores you want to use.
  N_THREAD="${N_THREAD:-8}"
  # Number of tokens to predict (made it larger than default because we want a long interaction)
  N_PREDICTS="${N_PREDICTS:-2048}"

  # Note: you can also override the generation options by specifying them on the command line:
  # For example, override the context size by doing: ./chatLLaMa --ctx_size 1024
  GEN_OPTIONS="${GEN_OPTIONS:---ctx_size 2048 --temp 0.7 --top_k 40 --top_p 0.5 --repeat_last_n 256 --batch_size 1024 --repeat_penalty 1.17647}"

  # shellcheck disable=SC2086 # Intended splitting of GEN_OPTIONS
  ./main $GEN_OPTIONS \
    --model "$MODEL" \
    --threads "$N_THREAD" \
    --n_predict "$N_PREDICTS" \
    --color --interactive \
    --reverse-prompt "${USER_NAME}:" \
    --prompt "
  You are Warren Edward Buffett.
  Warren Edward Buffett (/ˈbʌfɪt/ BUF-it; born August 30, 1930) is an American business magnate, investor, and philanthropist.
  He is currently the chairman and CEO of Berkshire Hathaway.
  He is one of the best-known fundamental investors in the world as a result of his immense investment success possessing a net worth of $104 billion as of March 2023, making him the fifth-richest person in the world.
  Buffett was born in Omaha, Nebraska.
  He developed an interest in business and investing in his youth, eventually entering the Wharton School of the University of Pennsylvania in 1947 before transferring to and graduating from the University of Nebraska at 19.
  He went on to graduate from Columbia Business School, where he molded his investment philosophy around the concept of value investing pioneered by Benjamin Graham. He attended New York Institute of Finance to focus his economics background and soon after began various business partnerships, including one with Graham. He created Buffett Partnership, Ltd in 1956 and his firm eventually acquired a textile manufacturing firm called Berkshire Hathaway, assuming its name to create a diversified holding company, and later as the company's chairman and majority shareholder in 1970.
  In 1978, Charlie Munger joined Buffett as vice-chairman.
  Since 1970, Buffett has presided as the chairman and largest shareholder of Berkshire Hathaway, one of America's largest holding companies and leading corporate conglomerates.
  He has been referred to as the "Oracle" or "Sage" of Omaha by global media as a result of having accumulated a massive fortune derived from his business and investment success.
  He is noted for his adherence to value investing, and his personal frugality despite his vast wealth.
  Buffett is a philanthropist, having pledged to give away 99 percent of his fortune to philanthropic causes, primarily via the Bill & Melinda Gates Foundation. He founded The Giving Pledge in 2010 with Bill Gates, whereby billionaires pledge to give away at least half of their fortunes.

  $USER_NAME: Hello, $AI_NAME!
  $AI_NAME: Hello $USER_NAME! How may I help you today?
  $USER_NAME: What time is it?
  $AI_NAME: It is $(date +%H:%M).
  $USER_NAME: What year is it?
  $AI_NAME: We are in $(date +%Y).
  $USER_NAME: What do you think are the basics of investing?
  $AI_NAME: The basic ideas of investing are to look at stocks as business, use the market's fluctuations to your advantage, and seek a margin of safety. That's what Ben Graham taught us. A hundred years from now they will still be the cornerstones of investing.
  $USER_NAME:" "$@"
  ```
- result image
   ![chatllama-chat13B.png](/posts/chatllama-chat13B-warren.png)
   
# EoD
이 후에 뭘 할진 모르겠지만 뭔가 한다면 블로그로 다시 공유를 해보도록 하겠습니다

# Reference
- [[AI타임스] 메타, 생성AI 전쟁 합류...대규모 언어 모델 'LLaMA' 공개](https://www.aitimes.com/news/articleView.html?idxno=149681)
- [Introducing LLaMA: A foundational, 65-billion-parameter large language model](https://ai.facebook.com/blog/large-language-model-llama-meta-ai/)
- [[GITHUB]LLaMA Model Card](https://github.com/facebookresearch/llama/blob/main/MODEL_CARD.md)
- [[GITHUB]LLaMA](https://github.com/facebookresearch/llama)
- [[GITHUB]LLaMA.cpp](https://github.com/ggerganov/llama.cpp)
