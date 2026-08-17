---
title: "아프리카TV 프로그램 분석 (1) - 초기 네트워크 설정"
categories: [Hacking, Reversing]
tags: [AfreecaTV, 리버싱, reversing, grid-delivery, 그리드-딜리버리, network-analysis, streaming, 스트리밍, P2P, 프로그램분석]
date: 2023-12-08 03:11:12 +0900
description: 아프리카TV의 고화질 스트리머(그리드 딜리버리) 프로그램이 설치 후 어떤 프로세스를 띄우고, 브라우저·Gate·Center 서버와 어떤 순서로 네트워크를 구성하는지 정리했습니다.
pin: false
---

> 이 글은 이전에 [티스토리 블로그](https://blackcon.tistory.com/411)에 작성했던 포스팅을 옮겨온 것입니다.
{: .prompt-info }

오늘은 갑자기 호기심이 생긴 아프리카TV의 프로그램을 분석해보았습니다. 좀 더 정확히 말하자면, 아프리카TV의 영상을 1080P 고화질로 볼 수 있도록 하는 **고화질 스트리머**라는 프로그램인데요. 이 고화질 스트리머라는 것은 [그리드 딜리버리 프로그램](https://m.blog.naver.com/ohbanson/221584081486)이라고 하여, 사용자들간의 데이터를 교류하며 딜레이 및 화질을 개선하기 위한 프로그램이라고 하는데요. 과연 어떤식으로 수행되는지 궁금하여 분석을 좀 시도해보았습니다.

최종 목표는 바이너리 분석까지 진행하고 내부에서 어떤 일이 일어나는지 작성하는 것이었으나, 아무래도 본업이 있는지라 많은 시간을 투자하진 못했구요. 프로세스 상태와 네트워크 변화들을 중심으로 분석한 내용들이니 참고하여 재밌게 봐주세요. **(+기술적인 내용은 별로 없음..)**

# 1. 프로세스

가장 먼저 프로그램을 설치한 후 process list를 확인해보면 아래와 같이 `afreecatvpackage.exe`가 실행되고 있는 것을 볼 수 있습니다. 이뿐 아니라 웹 브라우저에서 고화질로 동영상을 재생할 경우, 추가으로 afreecatvstreamer.exe라는 프로세스가 하나 더 실행되는데요. 이 프로세스가 핵심요소로 보여집니다.

## 1) afreecatvpackage.exe

- PATH: %HOMEPATH%\AppData\Local\afreeca\afreecatvpackage.exe
- Process List
  ![](/posts/afreecatv-analysis-1/afreeca1-01.png)
- Listening
  ![](/posts/afreecatv-analysis-1/afreeca1-02.png)

## 2) afreecatvstreamer.exe

- PATH: %HOMEPATH%\AppData\Local\afreeca\afreecatvstreamer.exe
- Process List
  ![](/posts/afreecatv-analysis-1/afreeca1-03.png)

## 3) 기타 설치된 프로그램들

- PATH: `C:\Users\사용자\AppData\Local\afreeca\`
- List
  ![Afreeca TV 설치파일](/posts/afreecatv-analysis-1/afreeca1-04.png)
  _Afreeca TV 설치파일_

# 2. 네트워크

글로 설명하기엔 너무 복잡하여 네트워크 연결 및 수행하는 작업을 그림으로 순차적으로 그렸습니다.

## 1) 네트워크 초기 설정

![Local Network](/posts/afreecatv-analysis-1/afreeca1-05.png)
_Local Network_

- 브라우저가 Client가 되며, afreecatvpackage.exe 프로세스가 Web socket 서버가 되어 통신을 함
- 이 구간에서는 네트워크 연결을 위해 프로세스와 브라우저간의 초기 설정을 진행
- 또한 스트리밍에 핵심이 되는 afreecatvstreamer.exe 파일이 실행됨
- 참고
    - HTMLPLAYER_PORT: 웹 브라우저로 스트리밍 데이터를 전송하기 위한 포트
    - [HLS_PORT](https://www.cdnetworks.com/ko/media-delivery-blog/http-live-streaming/): HTTP Live Streaming 용 포트인데, 아프리카TV에서 언제 활용되는지 모름
    - [RTMP_PORT](https://growthvalue.tistory.com/178):Real Time Messaging Protocol 용 포트인데, 아프리카TV에서 언제 활용되는지 모름

## 2) 방송 정보 설정 및 로깅

![Afreeca Network](/posts/afreecatv-analysis-1/afreeca1-06.png)
_Afreeca Network_

- afreecatvpackage.exe 는 프록시 느낌으로 활용되어 지며,
- Afreeca Server 중에서도 Gate 및 Center 서버로부터 방송 정보를 주고받음
- 샘플 데이터
    - (Send )브라우저-> HTMLPLAYER_PORT
      ```bash
      {"SVC":40,"RESULT":0,"DATA":{"gate_ip":"218.xxx.xxx.xxx","gate_port":3456,"center_ip":"110.xxx.xxx.xxx","center_port":19000,"broadno":249417161,"cookie":"","guid":"xxxxxxxxx~~","cli_type":42,"passwd":"","category":"00570002","JOINLOG":"log\u0011\u0006&\u0006uuid\u0006=\u00064b05002dc64e43052a2dd6a8ba378d42\u0006&\u0006geo_cc\u0006=\u0006KR\u0006&\u0006geo_rc\u0006=\u000641\u0006&\u0006acpt_lang\u0006=\u0006ko\u0006&\u0006svc_lang\u0006=\u0006ko_KR\u0006&\u0006os\u0006=\u0006win\u0006&\u0006is_streamer\u0006=\u0006true\u0006&\u0006is_row_latency\u0006=\u0006false\u0006&\u0006is_rejoin\u0006=\u0006false\u0006&\u0006is_auto\u0006=\u0006false\u0006&\u0006is_support_adaptive\u0006=\u0006true\u0006&\u0006uuid_3rd\u0006=\u000xxxxxxxxa2dd6a8ba378d42\u0012liveualog\u0011\u0006&\u0006is_clearmode\u0006=\u0006false\u0012","BJID":"xxxxxxxx","fanticket":"xxxxxxxxxad6ea4b05c_xxxxxx_2xxxx1_html5","addinfo":"ad_lang\u0011ko\u0012is_auto\u00110\u0012","update_info":0}}
      
    - (recv) HTMLPLAYER_PORT -> 브라우저
      ```json
      {"DATA":{"cSkinFile":"ps_Afreeca","cUserId":"","iExpireTime":0,"iFreeEventType":0,"iIpAddr":0,"iKeepAliveTime":330,"iMode":2},"RESULT":0,"SVC":41}
      

# 3. To be continue..

처음 시작은 전체적인 flow만 심플하게 정리하고 끝내야지라는 마음이었지만, 작성하다보니 간단하게 줄일 수 있는 내용이 아니었네요. :(
이번 포스팅에서는 실제 방송 데이터를 받기 전 단계까지의 내용만을 다루었는데요. 분석한게 좀 더 남아있으니, 다음 포스팅에 이어서 나머지것들도 작성해보도록 하겠습니다.
