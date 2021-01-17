# Application Layer
- Message
- DNS 서버가 Domain -> 네트워크 주소로 변환한다.

# Transport Layer
- Segment
- 만약 데이터가 크다면, 데이터를 쪼개고, 이 순서를 위해 Header에 순서번호 기록
- 흐름제어, 혼잡 제어, 신뢰성 등을 제공할 수 있다(TCP)

# Network Layer
- Datagram
- 다른 네트워크로의 전송 route 설계
- IP Header에 sIP dIP가 구성되어있다.
## 라우팅
- 경로를 찾아가게 하는 과정
- static한 방법(라우팅 테이블을 수동으로), dynamic한 방법(라우팅 테이블을 상황에 따라 다른 route로 생성)이 존재

# DataLink Layer
- Frame
- 자신이 속한 네트워크 상에서 동작
- Ethernet Header에 sMAC dMAC이 구성되어 있다.
- 이미 ARP Table에 있는 Mac주소의 경우 dMAC으로 패킷 전달
## ARP
- 논리적 주소인 IP 주소를 통해 물리적 주소인 Mac Address를 찾기 위한 프로토콜
- Destination IP Address를 통해 Destination Mac Address를 탐색하는 과정
- broadcasting을 통해 모든 인접 노드들에 요청을 보내는 것을 반복해서 Target을 찾는다.
- 이 때 이 broadcast를 매번 할 수 없으므로 만들어놓은 cache map이 ARP TABLE
- 이를 통해 어디로 보낼지를 판단한다.
# Physical Layer
- 실제 데이터 전송 매체
