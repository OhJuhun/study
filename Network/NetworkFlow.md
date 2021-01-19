# Application Layer
- Message
- DNS 서버가 Domain -> 네트워크 주소로 변환한다.

# Transport Layer
- Segment
- 만약 데이터가 크다면, 데이터를 쪼개고, 이 순서를 위해 Header에 순서번호 기록
- 흐름제어, 혼잡 제어, 신뢰성 등을 제공할 수 있다(TCP)
- 여기서부터는 End-System에서만 구현됨
## 흐름제어
- 송, 수신 데이터 처리 속도 해결
## 혼잡제어
- 송신측 데이터 전송, 네트워크 처리 속도 차이 해결

# Network Layer
- Datagram
- `다른 네트워크로의 전송` route 설계
- 라우터에는 여기까지만 구현이 되어 있다.
- IP Address는 논리적 주소
## header
- ToS : QoS를 위해 를 가지고 있고 이를 통해
- TTL : Time To Live 네트워크를 지날 때 마다 1 씩 감소(Loop 등으로 인해 지연시 Drop)
- Destination IP : 도착지 IP 주소
- Source IP : 출발지 IP 주소
## 포워딩 (Data Plane)
- 패킷 도착시 적절한 출력 링크로 이동시킨다

## 라우팅 (Control Plane)
- 경로를 찾아가게 하는 과정
- static한 방법(라우팅 테이블을 수동으로), dynamic한 방법(라우팅 테이블을 상황에 따라 다른 route로 생성)이 존재

# DataLink Layer
- Frame
- `자신이 속한 네트워크 상`에서 동작
- Ethernet Header에 sMAC dMAC이 구성되어 있다.
- 이미 ARP Table에 있는 Mac주소의 경우 dMAC으로 패킷 전달
- Mac Address는 물리적 주소
## ARP
- 논리적 주소인 IP 주소를 통해 물리적 주소인 Mac Address를 찾기 위한 프로토콜
- Destination IP Address를 통해 Destination Mac Address를 탐색하는 과정
- broadcasting을 통해 모든 인접 노드들에 요청을 보내는 것을 반복해서 Target을 찾는다.
- 이 때 이 broadcast를 매번 할 수 없으므로 만들어놓은 cache map이 ARP TABLE
- 이를 통해 어디로 보낼지를 판단한다.
### 동작과정
1. 외부 패킷 수신
1. ARP Cache Table 확인
1. 정보가 없으면 해당 Message Broadcast -> 이후 ARP Cache Table을 채움
1. 해당하는 호스트가 자신의 Mac주소==DMac 주소이면 Unicast로 다시 전송
# Physical Layer
- 실제 데이터 전송 매체
