---
title: "아프리카TV 프로그램 분석 (3) - P2P 패킷 구조 분석"
categories: [Hacking, Reversing]
tags: [AfreecaTV, 리버싱, reversing, P2P, packet-analysis, 패킷분석, protocol, Wireshark, PoC, streaming, 프로그램분석]
date: 2023-12-21 23:14:28 +0900
description: 아프리카TV 고화질 스트리머의 Child-Parent P2P 통신 흐름과 4개 Command의 헤더/바디 패킷 구조를 분석하고, 영상 데이터를 직접 요청하는 PoC까지 정리했습니다.
pin: false
---

> 이 글은 이전에 [티스토리 블로그](https://blackcon.tistory.com/413)에 작성했던 포스팅을 옮겨온 것입니다.
{: .prompt-info }

요즘 프로그램 분석하는게 취미가 된것마냥, 여가 시간이 나면 계속 분석만 하고 있네요 ㅎㅎ 벌써 아프리카TV 프로그램 분석 주제로 세번째 포스팅을 하게 되었습니다!

이번 포스팅의 주제는 P2P 패킷에 대해 분석한 내용을 다루었습니다. 나름의 Handshake 과정이 있었으며, 이번 분석으로 실제 미디어 파일을 받기까지 어떠한 작업이 이루어졌는지 알 수 있었는데요. 프로그램이 좀 방대하지만 이 포스팅에서는 `720p 영상`에 한하여 분석한 내용이 있으니 참고하셔서 봐주세요 :D

# 1. 통신 순서

이 프로그램에서 P2P 통신을 할 때 나름의 명칭이 있었는데요.(물론, 다른 P2P에서도 동일한 네이밍을 쓸수도 있음)

데이터를 요청하는 시청자를 Child, 영상 데이터를 제공해주는 시청자를 Parent라고 합니다. 나의 역할이 무엇인지에 따라서 Child와 Parent로 나뉘지만, 이 글에서는 Child 입장에서 글을 풀어가겠습니다.

Child 시청자는 인터넷 방송을 보기위해 방송을 들어갑니다. 이 때는 [아프리카TV 프로그램 분석 (2) - Grid Network](/posts/afreecatv-analysis-2-grid-network/) 글에서 다루었다시피, 각종 정보를 주고받고 Parent시청자의 정보도 획득하게 되는데요. Parent 시청자 정보를 받게된 후에는 아래 그림과 같은 Flow로 통신이 이루어지며, 실시간 스트리밍 영상을 전달받게 됩니다.

![Communication with ParentHost](/posts/afreecatv-analysis-3/afreeca3-01.png)
_Communication with ParentHost_

각 Flow 별 설명이 간단히 풀어보면 아래와 같습니다. 참고로 Command로 적힌 항목은 이 프로그램에서 정의해둔 2Byte 정수인데요. 아프리카TV 프로그램을 분석하다보면 매우 다양한 Command가 있지만, 기본적인 통신에 필요한 아래 4개의 Command만 다루겠습니다.

## 1) V2S_REQ_BROADCAST_STREAM_VER2

- 통신 흐름: Child (me) -> Parent
- Command: 0xCB2E
- 설명: 상대 사용자(Parent)에게 P2P Stream 데이터 요청합니다.

## 2) P2P_ACK_BROADCAST_STREAM_RIGHT_VER2

- **통신 흐름**: Parent -> Child (me)
- **Command**: 0x1BC7
- **설명**: Child(me) 요청에 대한 응답으로써, 스트리밍의 프레임 정보를 Child(me)에게 전달합니다.

## 3) V2S_REQ_CACHE_DATA_VER2

- **통신 흐름**: Child (me) -> Parent
- **Command**: 0xCB30
- **설명**: 재생에 필요한 프레임 정보를 상대 사용자(Parent)에게 요청합니다.

## 4) S2V_REP_CACHE_DATA_VER2

- **통신 흐름**: Parent -> Child (me)
- **Command**: 0xC748
- **설명**: 실제 스트리밍 미디어 데이터를 Child(me)에게 전달합니다.

# 2. 패킷 뜯어보기

앞서 다루었던 각 Flow의 패킷을 조금 더 세밀하게 뜯어보도록 하겠습니다. Child와 Parent는 일반적인 TCP 통신을 하며, 별도의 암호화 절차는 없었습니다. 그도 그럴것이 실시간 영상데이터인데 암복호화가 이루어지면 처리 시간이 평문일 때보다 더 걸릴 수도 있을 것 같네요. (혹시라도 실시간 영상에도 암호화가 이루어진다면 관련해서 댓글 부탁드립니다! 저도 이 분야는 처음이라..ㅎㅎ)

무튼 모든 데이터는 크게 Header와 Body로 구성되어 있는데요. Header는 0x10 byte이며, Body는 나머지 하위 데이터들입니다. 또한 Header의 구조는 모든 Command가 동일한 반면에 Body의 구조는 Command별로 다 달랐습니다! 그럼 각 항목별로 데이터를 소개드리겠습니다.

## 1) V2S_REQ_BROADCAST_STREAM_VER2 (ME -> PARENT)

- 패킷 캡쳐

![V2S_REQ_BROADCAST_STREAM_VER2](/posts/afreecatv-analysis-3/afreeca3-02.png)
_V2S_REQ_BROADCAST_STREAM_VER2_

- Header (파란색 박스)
  ```python
  Version Info (2byte): 0x202 # Default
  Command (2 byte): 0xCB2E # V2S_REQ_BROADCAST_STREAM_VER2
  Body Size (4 byte): 0x0000000c
  Check Sum (4byte): 0x0000C920 # checksum = version ^ command ^ body_size
  ??? (4byte): 0x19F928
  
- Body (빨간색 박스)
  ```python
  Broad Cast ID (4byte) : 0x0ee7cba1 # 생방송 ID
  Node Key (4byte) : 0x000014b8 # Child(me)의 ID
  Quality (4byte) : 0x00000001 # 화질 정보 (1=720p, 4=540p, 8=360p)
  

## 2) P2P_ACK_BROADCAST_STREAM_RIGHT_VER2 (PARENT -> ME)

- 패킷 캡쳐

![P2P_ACK_BROADCAST_STREAM_RIGHT_VER2](/posts/afreecatv-analysis-3/afreeca3-03.png)
_P2P_ACK_BROADCAST_STREAM_RIGHT_VER2_

- Header (파란색 박스)
  ```python
  Version Info (2byte): 0x202 # Default
  Command (2 byte): 0x1BC7 # P2P_ACK_BROADCAST_STREAM_RIGHT_VER2
  Body Size (4 byte): 0x00000018
  Check Sum (4byte): 0x000019DD # checksum = version ^ command ^ body_size
  ??? (4byte): 0x544c5553 # garbage data?
  
- Body (빨간색 박스)
  ```python
  Quality (4byte) : 0x00000001 # 화질 정보 (1=720p, 4=540p, 8=360p)
  Last Frame Number (new) (4byte) : 0x0030df54
  Last Frame Number (before) (4byte) : 0x00000000
  Last PTS (new) (4byte) : 0xc1c80d3b # PTS: 재생 시간 타임스탬프 (Presentation Timestamp)
  Last PTS (before) (4byte) : 0x00000610
  ??? (4byte) : 0x00000001 # 처리 로직이 안보이네요;
  

## 3) V2S_REQ_CACHE_DATA_VER2 (ME -> PARENT)

- 패킷 캡쳐

![V2S_REQ_CACHE_DATA_VER2](/posts/afreecatv-analysis-3/afreeca3-04.png)
_V2S_REQ_CACHE_DATA_VER2_

- Header (파란색 박스)
  ```python
  Version Info (2byte): 0x202 # Default
  Command (2 byte): 0xCB30 # V2S_REQ_CACHE_DATA_VER2
  Body Size (4 byte): 0x00000010
  Check Sum (4byte): 0x0000C922 # checksum = version ^ command ^ body_size
  ??? (4byte): 0x00D8D210
  
- Body (빨간색 박스)
  ```python
  Status (4byte) : 0x00000001 # 해상도?
     # {1: Original, 2: 2000K, 4: 1000K, 8: 500K, 0x10: 4000K, 0x20: 8000K}
  Last Frame number (new) (4byte) : 0x0030DF5C
  Last Frame number (before) (4byte) : 0x00000000
  ConnectStatus  (4byte): 0xFFFFFFFF # 최초연결: 0xffffffff, 기존연결: 0x00000006
  

## 4) S2V_REP_CACHE_DATA_VER2 (PARENT -> ME) // 영상데이터

- 패킷 캡쳐

![S2V_REP_CACHE_DATA_VER2](/posts/afreecatv-analysis-3/afreeca3-05.png)
_S2V_REP_CACHE_DATA_VER2_

- Header (파란색 박스)
  ```python
  Version Info (2byte): 0x202 # Default
  Command (2 byte): 0xC748 # S2V_REP_CACHE_DATA_VER2
  Body Size (4 byte): 0x0000047b
  Check Sum (4byte): 0x0000C131 # checksum = version ^ command ^ body_size
  ??? (4byte): 0x00000000
  
- Body (빨간색 박스)
  ```python
  Header (2byte) : 0x0001
  Size of header in body (2byte) : 0x0045
  Extention Size (4byte) : 0x00000000
  Raw data size (4byte): 0x00000436
  iFrameNo (4byte) : 0x0030df5c

  # 아래 부터는 부분적인 분석만 되어서 분석된 내용들만 등록합니다 ㅠㅠ
  Frame Type (1byte. offset: 0x28): 0x50 # {"A": "AAC", "I": "H264", "P": "H264"}
  media raw data (0x45 ~ Raw data size 만큼의 크기) # ts파일의 body만 있는듯 합니다. 
  				 	 	 # 영상으로 변환은 다음 포스팅에서 소개하겠습니다.
  

# 3. POC

끝으로, 위에서 분석한 내용을 토대로 Parent 사용자에게 영상 데이터를 요청하는 POC를 작성해보았습니다. 물론, 다른 시청자가 아니고 제가 아프리카 방송을 켜고 저의 PC와 VM만으로만 테스트 했습니다 :)

## 1) 테스트 환경 이미지

![Testing Environment](/posts/afreecatv-analysis-3/afreeca3-06.png)
_Testing Environment_

## 2) POC 결과 이미지

![POC](/posts/afreecatv-analysis-3/afreeca3-07.png)
_POC_
