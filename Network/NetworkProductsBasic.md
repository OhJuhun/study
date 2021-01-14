# L2 Switch
## 동작 원리
1. 자신의 Interface로 들어오는 Frame을 받아 Dmac 정보 확인
1. `자신의 Mac-addr table을 확인`
1. 내보내야할 Interface 결정 및 LookUp
1. Forwarding

## Mac-Address Table Create/Update
-  learning forwarding, flooding을 통해 mac-address table을 CU
### 과정
1. 1,2번 Input Port, 3번 Output Port
1. 최종 목표 Z
1. Mac Address Table이 비었을 때에는
1. source Mac을 보고 들어온 Interface와 Mapping하여 Mac Table을 채우고
1.  나에게 넘어온 Interface 외의 모든 Inteface에 Frame을 전달한다.
1. 1에서 들어왔을 때, 테이블이 비었다면 2로도 가고 L3로도 가게 된다.
![hel](./image/L2layer.png)

# L3 Router
- `서로 다른 네트워크` 연결(실제 End-To-End 통신, 다른 네트웍에 있더라도)

## 동작 원리
1. 다양한 L2,L1 Protocol 그에 적합한 Interface Type 제공
1. Frame에서 L2헤더를 떼고(DMac이 자신의 것이라면 == 내가 수신자다)
1. IP의 DIP정보를 통해  자신의Routing Table을 확인
1. Nexthop IP와 내보내야 할 Interface 결정
1. L2헤더 추가하여 Forwarding 

## 기능
### Routing Table Create / Update
- 들어오는 요청에 대해 어느 쪽으로 내보낼까에 대한 정보를 가진 Table
### ACL
- 해당 요청을 받을지 말지를 정함
### QoS


# L4 Switch
## 목적
- End Host 부하 분산을 위한 다중화 목적으로 이용 -> DataCenter 서버팜(?)
- 장비가 호스트처럼 동작하기 때문에 이중화를 해야 함
## 동작과정
1. Frame을 받아 L2(Ethernet) Header 제거(Dmac==자신)
1. L3(IP) Header 제거
1. L4 Header를 보고 어떤 서비스인지 판단
1. 어디로 어떻게 보내야할 지 결정