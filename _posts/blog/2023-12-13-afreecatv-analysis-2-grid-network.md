---
title: "아프리카TV 프로그램 분석 (2) - Grid Network"
categories: [Hacking, Reversing]
tags: [AfreecaTV, 리버싱, reversing, grid-network, 그리드네트워크, Frida, 후킹, hooking, P2P, little-endian, network-analysis, 프로그램분석]
date: 2023-12-13 00:26:17 +0900
description: 아프리카TV 고화질 스트리머가 Center 서버와 주고받는 데이터, Grid Network의 Parent IP/Port 리틀엔디언 인코딩, 그리고 Frida로 Parent 정보를 후킹해 뽑아내는 스크립트를 다룹니다.
pin: false
---

> 이 글은 이전에 [티스토리 블로그](https://blackcon.tistory.com/412)에 작성했던 포스팅을 옮겨온 것입니다.
{: .prompt-info }

지난 포스팅에 이어서 **고화질 스트리머 프로그램** 분석글을 작성하겠습니다. 이 전 [포스팅](/posts/afreecatv-analysis-1-network-init/)에서는 스트리밍 데이터를 받기 전에 어떠한 작업들이 이루어지는지 알아보았는데요. 이번 포스팅에서는 영상 데이터를 어디서 받아오는지를 중심으로 내용을 구성해보았습니다. 또한 Frida로 바이너리를 후킹하여 원하는 정보를 쉽게 받아오는 스크립트도 함께 소개드리겠습니다.

# 1. Client와 Center 서버간의 통신

이 전 포스팅에서 **afreecastreamer.exe**를 실행하고, 실시간 방송을 재생하면 Center로 불리는 아프리카 서버 (TCP PORT:19000)로 연결되는 것을 확인하였습니다. 이 Center와 연결한 후에는 다양한 작업을 하지만, 눈에 띄는 몇 가지만 나열해보면 아래와 같습니다.

## 1) Client -> Center

- 사용자의 각종 설정 값을 전달 함
- UUID, OS, Language 등을 비롯한 기본 정보도 전달되며,
- path_key, ticket과 같이 Center와 연결에 필요한 식별 정보도 함께 전달됨
- 참고) UUID와 path_key가 동일한 값
  ![Send data from Client to Center](/posts/afreecatv-analysis-2/afreeca2-01.png)
  _Send data from Client to Center_

## 2) Center -> Client

- 1440p 지원여부, 인증정보 등 방송 정보를 전달하며,
  ![Send information from Center to Client (1)](/posts/afreecatv-analysis-2/afreeca2-02.png)
  _Send information from Center to Client (1)_
- 실시간 방송의 채팅방 정보(broadno, chat IP, chat Port, chat roomno, title)를 전달함
  ![Send information from Center to Client (2)](/posts/afreecatv-analysis-2/afreeca2-03.png)
  _Send information from Center to Client (2)_
- Parent IP, Parent Port라는 정보를 전달함 (다음 목차에서 Detail하게 다룰 내용)
    - 아래 이미지는 최초로 제공되는 Parent IP, Port 이며, 스트리밍 데이터를 제공해주는 Host
      ![Main parent](/posts/afreecatv-analysis-2/afreeca2-04.png)
      _Main parent_
    - 추가로 3개의 Parent IP와 Port를 제공 받으며, 아마도 예비용 Parent 정보로 보여집니다.
      ![Sub parents](/posts/afreecatv-analysis-2/afreeca2-05.png)
      _Sub parents_

# 2. Client와 Client간의 통신

## 1) Grid Network

**Grid Network**라고 함은 쉽게 말해 사용자들끼리의 네트워크를 연결하여 형성한 것을 의미합니다. 보통의 인터넷 서비스들은 Server와 Client와 같이 1대 다수가 연결하는 망구조 인데요. 스트리밍 서비스에서 이런 네트워크 구조를 가진다면 Server에 적지않은 부담이 갈 것이고, 흔히 말하는 버퍼링이 발생할 가능성이 높아집니다.
하여, 아프리카TV를 포함한 각종 라이브 스트리밍 서비스에서는 Grid Network를 체택하였는데요.(Grid 프로그램 설치시에만 구성됨) 이는 동일한 라이브 스트리밍 영상을 보는 사용자들끼리 Live Cache(또는 Streaming Cache) 데이터를 공유하도록 함으로써, 서버의 부담을 줄이면서 사용자에게 빠른 영상과 화질을 제공해줄 수 있게 됩니다.

## 2) 스트리밍 영상 제공자

### A) Parent IP와 Port에 대하여

우선 위에서 잠깐 나온 Parent IP, Parent Port라는 키워드부터 짚고 가겠습니다. 이 키워드는 아프리카TV의 개발자분들이 정의해둔 이름으로 보입니다. 프로그램을 분석하면서 알게된 결과, **ParentIP**는 실시간 영상 정보를 `나`에게 전달해주는 노드이며, **Parent Port**는 이 노드와 연결할 포트입니다. 이러한 IP와 Por 정보는 AfreecaTV의 `Center`로부터 전달받게 되는데요. Center의 내부 로직이 어떻게 되어있는진 모르겠으나, 아마도 같은 라이브 방송을 시청하고 있는 다른 사용자의 IP와 Port를 전달해주는 듯 합니다.

그도 그럴것이 Parent Port의 범위는 보통 `10000 ~ 100xx`로 보여지는데요. 저의 PC에서도 afreecastreamer.exe를 실행하니, 동일한 대역의 Port를 열고 Listening하는 것을 확인할 수 있었습니다.

![Listen ports of Afreecastreamer.exe](/posts/afreecatv-analysis-2/afreeca2-06.png)
_Listen ports of Afreecastreamer.exe_

### B) Parent IP, Port의 형태 변환

분석 초반에 Network Packet에서 Parent IP와 Port를 접하게 되었습니다. 제공되는 형태는 JOSN 포맷에 묶여져서 전달되었으며, IP와 Port의 Type은 10진수로 전달되었는데요. 전달받은 10진수를 IP형태로 변환을 해보아도 실제 통신을 하고있지 않은 뜬금없는 IP가 나왔으며, Port 또한 전혀 생뚱많은 값이었습니다. 이 값들은 AfreecaTV의 Center에서 IP/Port 정보를 Little Endian 방식으로 변환한 후에 각 Client에게 제공해주고 있었는데요.

우선 Wireshark로 실제 패킷을 제공해주고 있는 Parent IP와 Port를 보면 IP는 `222.109.223.157`이고 Port는 `10030`인것을 알 수 있습니다.

![Receive data from parent host](/posts/afreecatv-analysis-2/afreeca2-07.png)
_Receive data from parent host_

그리고 아래 데이터는 포스팅 초반에 언급되었던 Parent IP와 Parent Port인데요. 아래의 형태로 Center 서버로부터 전달받았으며, 10진수의 값들을 IP형태로 변환해본 코드입니다.

- Center로 부터 전달받은 데이터 ("parent_ip":2648665566, "parent_port":11815)
  ```json
  {"list":[{"cur_frame_no":29474472,"cur_pts":0,"cur_seq_audio":950753,"cur_seq_video":28523717,"parent_ip":2648665566,"parent_port":11815,"parent_sess":22266,"quality":1,"skip_frame":0}]}
  
- 단순 IP 및 Port 형태로 변환
  ```python
  ip = 2648665566
  port = 11815
  ip_1 = (ip & 0xff000000) >> 0x18
  ip_2 = (ip & 0xff0000) >> 0x10
  ip_3 = (ip & 0xff00) >> 0x8
  ip_4 = (ip & 0xff)
  print(f"Parent IP: {ip_1}.{ip_2}.{ip_3}.{ip_4}:{port}")

  # Outupt: Parent IP: 157.223.109.222:11815
  

예상과는 다른 결과가 나와서 놀랐습니다. 실제 네트워크 패킷에서의 Source IP는 `222.109.223.157`이고 Port는 `10030`이었지만, 10진수를 단순히 IP형태로 변환한 코드에서는 `157.223.109.222:11815`가 출력되었습니다.

눈치채신분은 "아, IP가 거꾸로 출력되었네?"라고 인지가 되었을텐데요. 추측컨데 이는 Center에서 Parent 정보를 Client에게 전달해줄 때 Little Endian 방식으로 그대로 전달을 해준듯하구요. Port도 Little Endian 일것이라고 판단하여, 아래의 스크리립트로 다시 변환을 해보니 원하는 Parent IP와 Port를 획득할 수 있었습니다.

- Little Endian형태의 10진수를 IP및 Port 형태로 변환

```python
import struct 
parent_ip = 2648665566 
parent_ip = struct.pack("<I", parent_ip) 
parent_ip = struct.unpack(">I",parent_ip)[0] 
ip_1 = (parent_ip & 0xff000000) >> 0x18 
ip_2 = (parent_ip & 0xff0000) >> 0x10 
ip_3 = (parent_ip & 0xff00) >> 0x8 
ip_4 = (parent_ip & 0xff) 
print(f"Parent IP: {ip_1}.{ip_2}.{ip_3}.{ip_4}") 

parent_port = 11815 
parent_port = struct.pack("<I", parent_port)[:2] 
parent_port = struct.unpack(">h", parent_port)[0] 
print(f"Parent Port: {parent_port}") 
''' 
# Output
Parent IP: 222.109.223.157 
Parent Port: 10030 
'''

### C) Frida를 이용한 Parent IP, Port 정보 획득

그럼 이 Parent IP와 Parent Port에 대한 정보를 좀 더 손 쉽게 구할 방법을 고안하다가, **Frida**를 이용하여 afreecastreamer.exe 메모리에서 받아서 출력해주는 방식을 생각해보았습니다. 분석을 좀 용이하게 하기 위해서 작성한 Script이며, "이런 방식으로 정보를 수집할수 있다"정도의 참고로만 봐주세요.

- 후킹할 지점 (module: NetControl.dll, offset: 0xe5dd)
    - Disassembly
      ![](/posts/afreecatv-analysis-2/afreeca2-08.png)
    - Hex-ray
      ![](/posts/afreecatv-analysis-2/afreeca2-09.png)
- Frida Code

```python
import frida, sys
import struct

def get_parent_info():
    # Frida JavaScript 코드
    script_code = """
    // 모듈과 함수 이름
    var moduleName = 'NetControl.dll';
    var offset = 0xe5dd;

    // 모듈 가져오기
    var module = Process.getModuleByName(moduleName);
    var hook_addr = module.base.add(offset);

    // 후킹
    Interceptor.attach(hook_addr, {
        onEnter: function (args) {
            var stack = this.context.esp;
            var ip = Memory.readPointer(stack.add(0x50), 4);
            var port= Memory.readPointer(stack.add(0x54), 2);
            send({ parent_ip: ip, parent_port: port });
        },
        onLeave: function (retval) {
        }
    });
    """
    return script_code
    
def on_message( message, data ):
    payload = message["payload"]
    if "parent_ip" in payload:
        # parent_ip, parent_port가 little endian으로 지정되어 있어서, 비트연산을 해주어야 원래 값이 나온다.
        parent_ip = int(payload["parent_ip"], 16)
        parent_port = int(payload["parent_port"], 16)
        parent_port = struct.pack("<I", parent_port)[:2]
        parent_port = struct.unpack(">h", parent_port)[0]
        print( f"[Grid Node] {parent_ip&0x000000ff}.{(parent_ip&0x0000ff00)>>0x8}.{(parent_ip&0x00ff0000)>>0x10}.{(parent_ip&0xff000000)>>0x18}", parent_port)

def main():
    process = "afreecatvstreamer.exe"
    session = frida.attach( process )
    fsc = get_parent_info() 
    script = session.create_script( fsc)

    script.on( "message", on_message )
    script.load()
    print("=======================================")
    print("[!] Ctrl+D on UNIX, Ctrl+Z on Windows/cmd.exe to detach from instrumented program.\n\n")
    sys.stdin.read()
    session.detach()
    
if __name__ == "__main__":
    main()

- 실행 결과

![Get parent info using frida script](/posts/afreecatv-analysis-2/afreeca2-10.png)
_Get parent info using frida script_

# 3. 마치며

핳, 기술글을 시리즈로 쓰는것도 쉽지 않은 작업이네요. 마음같아서는 스트리밍 동영상을 처리하는 로직을 다음편에 작성하고 마무리 하고 싶은데요. 아직 분석이 완벽히 끝난게 아니라, 언제 또 작성할진 모르겠습니다! 무쪼록 기회가 된다면 이어서 써보도록 하며, 긴 글 읽어주셔서 감사드립니다 :)
