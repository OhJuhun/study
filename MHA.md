# MHA
- Master DB가 장애로 서비스가 불가한 상태가 되면 자동 failover, slave DB > Master DB로 승격
- `서비스 다운 타임을 최소화 하는 auto failover Solution`
- High Availability를 보장할 수 있다.

## 구성
- MHA Manager, Master, Slave(Master-Slave 구조로만 사용하기도 함-> 이 때는 slave에 manager 설치)
- Master - Slave는 하나의 VIP 공유 > 해당 VIP를 이용하여 장애 발생시 VIP를 통해 조작

## 장점
- 최소한의 다운 타임으로 Master 장애 처리
- Master에 장애가 발생해도 Master/Slave의 데이터 불일치가 발생하지 않음
- 현재 DB Server 설정을 변경할 필요가 없음
- Clustering에 비해 Server가 많이 존재할 필요가 없다.

## 장애 체크
1. MHA Manager가 Master DB를 주기적으로 Connect, Select, Insert.
1. N회 실패시 장애로 인식 및 Failover 수행

## Java Spring MHA
- 위와 같이 설정되어있는 상태의 Connection Pool에서 `ReadOnly=true`인 Transaction의 경우 Slave와 Connection, `false`일 경우 Master와 Connection
- 이걸 Spring Transactional에서 처리해주는 것인, MHA Manager에서 처리해주는 것인지 확실치 않음

## Python SQLAlchemy MHA